---
Title: Statistics Meet Cryptanalysis
Description: Explore the use of the Sequential probability ratio test (SPRT) and Log-Likelihood Ratios (LLR) for filtering 60% probability bit-flip noise in a black-box padding oracle attack. This article breaks down the entire process with strong mathematical foundations and real-world connections.
slug: stats-meet-crypto
date: 2026-09-13 00:00:00+0000
image: cover.png
categories:
    - Cryptography
tags:
    - crypto
    - cryptanalysis
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## Introduction

The padding oracle attack is a fundamental vulnerability affecting symmetric block ciphers. It occurs when an attacker possesses a ciphertext and has access to a decryption endpoint (the oracle) that reveals whether the decrypted payload has valid padding. In a real-world scenario, a threat actor intercepts a ciphertext transmitted between two servers, maliciously modifies it, and forwards it to the destination. By observing the server's responses—such as distinct error messages or log entries indicating a padding failure—the attacker can systematically decrypt the entire message byte-by-byte. 

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/oracle-attack.gif" width="600" height="450">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 1: Complete padding oracle attack.
  </figcaption>
</figure>

Furthermore, we must account for noise in the oracle-attacker channel, which can distort the server's response regarding padding validity. In this scenario, we focus exclusively on bit-flip noise, where the boolean response of the oracle (e.g., 1 for valid padding, 0 for invalid padding) is inverted during transmission. Continuing with the previous example, this can occur when a Web Application Firewall (WAF) or rate limiter occasionally drops the server's HTTP 200 OK (valid padding) response, causing the attacker's automated tooling to misclassify a network timeout as a padding error (an artificial 0). 

To simulate this environment, I developed a server that accepts a plaintext message and encodes it in hexadecimal. To replicate the right-to-left dependency inherent in block cipher decryption attacks, a secondary endpoint evaluates byte guesses. The oracle enforces a strict chaining rule: it will only evaluate the current byte if all preceding bytes (from right to left) are guessed correctly. This underlying logic yields four foundational states:

1. Guess is correct, but preceding bytes are incorrect → **Incorrect**
2. Guess is correct, and preceding bytes are correct → **Correct**
3. Guess is incorrect, and preceding bytes are incorrect → **Incorrect**
4. Guess is incorrect, but preceding bytes are correct → **Incorrect**

To model the noisy channel, a bit-flip is introduced. There is a 60% probability that the oracle's true boolean response is inverted before reaching the attacker. At first glance, the system appears heavily biased toward "incorrect" responses, as three of the four baseline states result in failure. However, in practice, only the scenarios where the preceding bytes are correct remain relevant. By employing robust statistical algorithms, we can determine the preceding sequence with high confidence, effectively reducing the problem space to a binary evaluation of the current byte. 

In this post, we will explore three different algorithms, ordered from lowest to highest expected performance: Majority Vote, Log-Likelihood Ratio (LLR), and Thompson Sampling.

## Recognition of the oracle

The oracle interacts via a simple message structure: target_index,guessed_hex_byte,guessed_suffix. Here, target_index can also accept negative values to facilitate traversal. For pedagogical simplicity, the SECRET is represented in hexadecimal form. This reduces the search space for each character from the 256 possibilities of a standard byte down to 16 (0–9, a–f), allowing us to illustrate the core concepts more clearly.

```python
import sys
import random

# Hardcoded secret in hexadecimal ("secret_message")
SECRET = "7365637265745f6d657373616765"
# Noise probability (60% chance to flip the boolean result)
NOISE_P = 0.6

def main():
    # Continuous loop reading from standard input
    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        
        try:
            # Parse the incoming CSV format: index,guess,suffix
            parts = line.split(',')
            target_index = int(parts[0])
            guessed_hex_byte = parts[1]
            guessed_suffix = parts[2] if len(parts) > 2 else ""
            
            # 1. Slice the true secret to get the suffix strictly after the target index
            true_suffix = SECRET[target_index + 1:]
            
            # 2. Strict right-to-left dependency check and byte
            # validation
            if len(parts) > 2:
                if guessed_suffix != true_suffix:
                    base_result = 0
                else:
                    true_byte = SECRET[target_index]
                    if guessed_hex_byte == true_byte:
                        base_result = 1
                    else:
                        base_result = 0
            else:
                true_byte = SECRET[target_index]
                if guessed_hex_byte == true_byte:
                    base_result = 1
                else:
                    base_result = 0
                    
            # 3. Noise Injection: Bit-flip based on probability p
            if random.random() < NOISE_P:
                base_result = 1 - base_result
                
            # Output to stdout and flush immediately to prevent pipe deadlocks
            sys.stdout.write(f"{base_result}\n")
            sys.stdout.flush()
            
        except Exception:
            # Fallback for malformed input
            sys.stdout.write("0\n")
            sys.stdout.flush()

if __name__ == "__main__":
    main()
```

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/oracle-last-byte-tests.png" width="500" height="350">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 2: Some guesses over the last byte "5".
  </figcaption>
</figure>

## Bias measurement

To determine the server's noise bias, we can approach the problem using two distinct methods. The first is a white-box approach: inspecting the server's source code, assuming it is available. The second, more versatile approach is empirical black-box analysis. By querying the endpoint with a set of test characters—knowing that random guesses will predominantly fail—we can analyze the resulting response distribution to accurately measure the channel's underlying noise parameters. Below is a bias measurement tool that tests three arbitrary bytes, leveraging a high volume of queries to maximize precision, as the primary objective is to estimate the noise floor rather than recover the plaintext. In this case, we will take the empirical approach by generating binomial distributions for three different hexadecimal characters—one correct ("5") and two incorrect ("1" and "a")—to visually compare and analyze their distributional differences.

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/bias-measurements.png" width="700" height="500">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 3: Bias measurement for 3 different last byte with 1000 experiments (N) and 1000 queries per experiment (k).
  </figcaption>
</figure>

From the empirical distributions, we can conclude that the channel's bit-flip probability is approximately 60%, while correct, uncorrupted responses average a 40% response rate. This precise quantification of the noise floor is critical and will serve as the mathematical foundation for our statistical recovery algorithms.

## Sequential probability ratio test (SPRT)

The Sequential Probability Ratio Test (SPRT) is just a framework that discards rigid, fixed-size sample collection in favor of dynamic, data-driven evaluation. By continuously accumulating evidence into a Log-Likelihood Ratio (LLR) after every individual observation, SPRT measures the cumulative weight against mathematically strict upper and lower boundaries tied directly to your desired Type I (alpha) and Type II (beta) error rates. Because the test halts execution the exact moment the score breaches either threshold, it achieves extraordinary sampling efficiency—drastically reducing the average number of required queries compared to traditional fixed-size tests—while rigorously guaranteeing your precision and error tolerances.

"The core mechanism of this test is maintaining a separate Log-Likelihood Ratio (LLR) for every possible character value. Since we are dealing exclusively with lowercase hexadecimal characters (0-9 and a-f), the fundamental question we must ask is: *'Is the byte at position X equal to 'a'?'*  If we were to run this query repeatedly, the results should converge toward one of these two distinct distributions:

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/guess-correct-likelihood.png" width="700" height="500">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 4: Guess 'a' for position X is correct.
  </figcaption>
</figure>

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/guess-incorrect-likelihood.png" width="700" height="500">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 4: Guess 'a' for position X is incorrect.
  </figcaption>
</figure>

Following the previous example of testing whether the character at position X is 'a', we can set up the hypothesis test with the null hypothesis H0 (it is 'a') and the alternative hypothesis HA (it is not 'a'). The intuition behind the likelihood ratio is that we can quantify how much more likely we are to get a 'No' response if the character at position X is indeed 'a' (expressed as P(No|a) / P(Yes|a)), which indicates how strongly we should lean toward H0. Conversely, we can quantify how much more likely we are to get a 'Yes' response (expressed as P(Yes|a) / P(No|a)), which tells us how strongly we should lean toward HA.

For each character test, we must account for two types of errors: a Type I error (rejecting a true H0, represented by alpha as a false positive) and a Type II error (accepting a false H0, represented by beta as a false negative). In statistics, the first step is to define our overall success probability and set the parameters of our test accordingly. For example, if we want an overall decryption success rate of 90%—and since each character extraction is an independent experiment—we need a per-character success chance of roughly 0.9^(1/28) = 0.9962.

This leaves a total failure budget of 1 - 0.9962 = 0.0038 to distribute across our checks. Dividing this across the 16 possible hexadecimal candidates per position gives a per-test error rate of 0.0038 / 16 = 0.0002375. Because each character evaluation can face Type I or Type II errors, defining alpha = beta = 0.0002375 ensures our error bounds remain mathematically rigorous across the entire key recovery process.

Wald's sequential analysis provides the mathematical formulas for the bounds we need to add or subtract for each response. Within the context of the Log-Likelihood Ratio—details of which are omitted here since excellent resources already cover the full mathematical background, though it is simply the natural logarithm applied to the likelihood ratio we just described—the upper bound for acceptance and the lower bound for rejection are defined as c2 = ln( (1 - beta) / alpha ) = 8.3451 and c1 = ln( beta / (1 - alpha) ) = -8.3451. Consequently, the score adjustments required for each response are a 'No' resulting in +ln(1.5) = +0.405, and a 'Yes' resulting in +ln(0.66) = -0.405.
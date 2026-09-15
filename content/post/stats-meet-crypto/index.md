---
Title: Statistics Meet Cryptanalysis
Description: Explore the use of Log-Likelihood Ratios (LLR) for filtering 60% probability bit-flip noise in a black-box padding oracle attack. This article breaks down the entire process with strong mathematical foundations and real-world connections.
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
  <img src="/p/stats-and-opti-meet-crypto/oracle-attack.gif" width="600" height="450">
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
  <img src="/p/stats-and-opti-meet-crypto/oracle-last-byte-tests.png" width="500" height="350">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 2: Some guesses over the last byte "5".
  </figcaption>
</figure>

## Bias measurement

To determine the server's noise bias, we can approach the problem using two distinct methods. The first is a white-box approach: inspecting the server's source code, assuming it is available. The second, more versatile approach is empirical black-box analysis. By querying the endpoint with a set of test characters—knowing that random guesses will predominantly fail—we can analyze the resulting response distribution to accurately measure the channel's underlying noise parameters. Below is a bias measurement tool that tests three arbitrary bytes, leveraging a high volume of queries to maximize precision, as the primary objective is to estimate the noise floor rather than recover the plaintext. In this case, we will take the empirical approach by generating binomial distributions for three different hexadecimal characters—one correct ("5") and two incorrect ("1" and "a")—to visually compare and analyze their distributional differences.

<figure style="text-align: center;">
  <img src="/p/stats-and-opti-meet-crypto/bias-measurements.png" width="700" height="500">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 3: Bias measurement for 3 different last byte with 1000 experiments (N) and 1000 queries per experiment (k).
  </figcaption>
</figure>

From the empirical distributions, we can conclude that the channel's bit-flip probability is approximately 60%, while correct, uncorrupted responses average a 40% response rate. This precise quantification of the noise floor is critical and will serve as the mathematical foundation for our statistical recovery algorithms.

## Log Likelihood Ratio
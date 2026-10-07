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
weight: 1
params:
  math: true
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

1. Guess is correct, but preceding bytes are incorrect -> **Incorrect**
2. Guess is correct, and preceding bytes are correct -> **Correct**
3. Guess is incorrect, and preceding bytes are incorrect -> **Incorrect**
4. Guess is incorrect, but preceding bytes are correct -> **Incorrect**

To model the noisy channel, a bit-flip is introduced. There is a 60% probability that the oracle's true boolean response is inverted before reaching the attacker. At first glance, the system appears heavily biased toward "incorrect" responses, as three of the four baseline states result in failure. However, in practice, only the scenarios where the preceding bytes are correct remain relevant. By employing robust statistical algorithms, we can determine the preceding sequence with high confidence, effectively reducing the problem space to a binary evaluation of the current byte. 

In this post, we will explore the SPRT algorithm, which uses log-likelihood ratios as its principal tool, with the approach to the objective proceeding step by step.

## Recognition of the oracle

The oracle interacts via a simple message structure: `target_index,guessed_hex_byte,guessed_suffix`. Here, `target_index` can also accept negative values to facilitate traversal. For pedagogical simplicity, the SECRET is represented in hexadecimal form. This reduces the search space for each character from the 256 possibilities of a standard byte down to 16 (0–9, a–f), allowing us to illustrate the core concepts more clearly.

```python
import sys
import random

# Hardcoded secret in hexadecimal ("secret_message")
SECRET = "7365637265745f6d657373616765"
# Noise probability (60% chance to flip the boolean result)
NOISE_P = 0.6

def main():
    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            parts = line.split(',')
            target_index = int(parts[0])
            guessed_hex_byte = parts[1]
            guessed_suffix = parts[2] if len(parts) > 2 else ""
            true_suffix = SECRET[target_index + 1:]
            if len(parts) > 2:
                if guessed_suffix != true_suffix:
                    base_result = 0
                else:
                    true_byte = SECRET[target_index]
                    base_result = 1 if guessed_hex_byte == true_byte else 0
            else:
                true_byte = SECRET[target_index]
                base_result = 1 if guessed_hex_byte == true_byte else 0
            if random.random() < NOISE_P:
                base_result = 1 - base_result
            sys.stdout.write(f"{base_result}\n")
            sys.stdout.flush()
        except Exception:
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
    Figure 3: Bias measurement for 3 different last bytes with 1000 experiments (N) and 1000 queries per experiment (k).
  </figcaption>
</figure>

From the empirical distributions, we can conclude that the channel's bit-flip probability is approximately 60%, while correct, uncorrupted responses average a 40% response rate. This precise quantification of the noise floor is critical and will serve as the mathematical foundation for our statistical recovery algorithms.

## Sequential probability ratio test (SPRT)

The Sequential Probability Ratio Test (SPRT) is a framework that discards rigid, fixed-size sample collection in favor of dynamic, data-driven evaluation. By continuously accumulating evidence into a Log-Likelihood Ratio (LLR) after every individual observation, SPRT measures cumulative weight against mathematically strict upper and lower boundaries tied directly to your desired Type I (\(\alpha\)) and Type II (\(\beta\)) error rates. Because the test halts execution the exact moment the score breaches either threshold, it achieves extraordinary sampling efficiency—drastically reducing the average number of required queries compared to traditional fixed-size tests—while rigorously guaranteeing your precision and error tolerances.

The core mechanism of this test is maintaining a separate Log-Likelihood Ratio (LLR) for every possible character value. Since we are dealing exclusively with lowercase hexadecimal characters (0-9 and a-f), the fundamental question we must ask is: *'Is the byte at position X equal to 'a'?'* If we were to run this query repeatedly, the results should converge toward one of these two distinct distributions:

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/guess-correct-likelihood.png" width="700" height="500">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 4: Guess 'a' for position X is correct.
  </figcaption>
</figure>

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/guess-incorrect-likelihood.png" width="700" height="500">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 5: Guess 'a' for position X is incorrect.
  </figcaption>
</figure>

Following the previous example of testing whether the character at position X is 'a', we can set up the hypothesis test with the null hypothesis \(H_0\) (it is 'a') and the alternative hypothesis \(H_A\) (it is not 'a'). The intuition behind the likelihood ratio is that we can quantify how much more likely we are to get a 'No' response if the character at position X is indeed 'a' (expressed as \(P(\text{No} \mid a) / P(\text{Yes} \mid a) = 0.6 / 0.4 = 1.5\)), which indicates how strongly we should lean toward \(H_0\). Conversely, we can quantify how much more likely we are to get a 'Yes' response (expressed as \(P(\text{Yes} \mid a) / P(\text{No} \mid a) = 0.4 / 0.6 = 0.66\)), which tells us how strongly we should lean toward \(H_A\).

For each character test, we must account for two types of errors: a Type I error (rejecting a true \(H_0\), represented by \(\alpha\) as a false positive) and a Type II error (accepting a false \(H_0\), represented by \(\beta\) as a false negative). In statistics, the first step is to define our overall success probability and set the parameters of our test accordingly. For example, if we want an overall decryption success rate of 90%—and since each character extraction is an independent experiment—we need a per-character success chance of roughly \(0.9^{1/28} \approx 0.9962\).

This leaves a total failure budget of \(1 - 0.9962 = 0.0038\) to distribute across our checks. Dividing this across the 16 possible hexadecimal candidates per position gives a per-test error rate of \(0.0038 / 16 = 0.0002375\). Because each character evaluation can face Type I or Type II errors, defining \(\alpha = \beta = 0.0002375 / 2 = 0.00011875\) ensures our error bounds remain mathematically rigorous across the entire key recovery process.

Wald's sequential analysis provides the mathematical formulas for the bounds we need to add or subtract for each response. Within the context of the Log-Likelihood Ratio—details of which are omitted here since excellent resources already cover the full mathematical background, though it is simply the natural logarithm applied to the likelihood ratio we just described—the upper bound for acceptance and the lower bound for rejection are defined as \[c_2 = \ln\left(\frac{1 - \beta}{\alpha}\right) = 9.0384\] and \[c_1 = \ln\left(\frac{\beta}{1 - \alpha}\right) = -9.0384\]. Consequently, the score adjustments required for each response are a 'No' resulting in \(\ln(1.5) = +0.405\), and a 'Yes' resulting in \(\ln(0.66) = -0.405\). 

The next step is to write the script to decrypt the text using only requests to the oracle.

```python
import subprocess
import random
import sys

ORIGINAL_SECRET = "7365637265745f6d657373616765"
SECRET_LENGTH = len(ORIGINAL_SECRET)
THRESHOLD = 9.0384

def run_single_experiment_debug():
    print(f"[*] Starting 1 debugging experiment...")
    print(f"[*] Target secret: {ORIGINAL_SECRET} (Length: {SECRET_LENGTH})\n")
    try:
        proc = subprocess.Popen(
            ['python', 'oracle.py'],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            text=True,
            bufsize=1
        )
    except Exception as e:
        print(f"[!] Critical error starting the oracle: {e}")
        return False

    recovered_secret = [""] * SECRET_LENGTH
    recovered_suffix = ""
    total_queries_overall = 0
    
    for target_index in range(SECRET_LENGTH - 1, -1, -1):
        true_byte_char = ORIGINAL_SECRET[target_index]
        print(f"\n" + "-"*50)
        print(f"[DEBUG] Decrypting byte at index [{target_index}] | Expected real character: '{true_byte_char}'")
        print(f"[DEBUG] Current accumulated suffix: '{recovered_suffix}'")
        
        pt_table = {c: 0.0 for c in "0123456789abcdef"}
        query_counts = {c: 0 for c in pt_table}
        active_chars = list(pt_table.keys())
        accepted_char = None
        iteration_step = 0
        
        while active_chars and not accepted_char:
            iteration_step += 1
            max_score = max(pt_table[c] for c in active_chars)
            tied_chars = [c for c in active_chars if pt_table[c] == max_score]
            guess_char = random.choice(tied_chars)
            
            query = f"{target_index},{guess_char},{recovered_suffix}\n"
            proc.stdin.write(query)
            proc.stdin.flush()
            
            query_counts[guess_char] += 1
            total_queries_overall += 1
            
            raw_response = proc.stdout.readline()
            if not raw_response:
                print(f"[!] The oracle closed the connection unexpectedly at index {target_index}.")
                break
                
            response = int(raw_response.strip())
            old_score = pt_table[guess_char]
            if response == 0:
                pt_table[guess_char] += 0.405
            else:
                pt_table[guess_char] -= 0.405
            new_score = pt_table[guess_char]
            
            print(f"[DEBUG Iter {iteration_step}] Testing char '{guess_char}' (Attempts: {query_counts[guess_char]}) | Oracle Resp: {response} | LLR: {old_score:.3f} -> {new_score:.3f}")
            
            if pt_table[guess_char] > THRESHOLD:
                accepted_char = guess_char
                print(f"[+] CHARACTER ACCEPTED! '{accepted_char}' at index [{target_index}] after {iteration_step} total queries.")
                break
                
            if pt_table[guess_char] < -THRESHOLD:
                if guess_char in active_chars:
                    active_chars.remove(guess_char)
                    print(f"[-] Character '{guess_char}' discarded by LLR < -{THRESHOLD}.")

        if not accepted_char and len(active_chars) == 1:
            accepted_char = active_chars[0]
            print(f"[+] CHARACTER ACCEPTED BY EXCLUSION! '{accepted_char}' at index [{target_index}].")

        if not accepted_char:
            print(f"[!] FAILURE! All characters were eliminated at index [{target_index}].")
            break

        print(f"[DEBUG] Query breakdown per character for index [{target_index}]: {query_counts}")
        recovered_secret[target_index] = accepted_char
        recovered_suffix = accepted_char + recovered_suffix
        print(f"[DEBUG] Partial secret state: {''.join([c if c != '' else '?' for c in recovered_secret])}")

    proc.stdin.close()
    proc.wait()
    
    hex_result = "".join([c if c != '' else '?' for c in recovered_secret])
    print("\n" + "="*50)
    print(f"Original Secret:      {ORIGINAL_SECRET}")
    print(f"Recovered Secret:     {hex_result}")
    print(f"Total Queries (All):  {total_queries_overall}")
    print(f"Partial/final suffix: '{recovered_suffix}'")
    print("="*50)
    
    if "".join(recovered_secret) == ORIGINAL_SECRET:
        print("[+] EXPERIMENT SUCCESSFUL! Exact match.")
        return True
    else:
        print("[-] EXPERIMENT FAILED.")
        return False

if __name__ == "__main__":
    run_single_experiment_debug()
```

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/attack-succesfull.png" width="400" height="300">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 6: Results of running the attack once.
  </figcaption>
</figure>

To validate the robustness of the implementation, a wrapper script was built to execute the entire attack across 10,000 independent trials, generating empirical performance metrics.

<figure style="text-align: center;">
  <img src="/p/stats-meet-crypto/attack-statistics-10000.png" width="400" height="300">
  <figcaption style="font-size: 0.9em; color: gray; margin-top: 5px;">
    Figure 7: Results of running the attack 10,000 times.
  </figcaption>
</figure>

The observed empirical success rate of \(98\%\), which surpasses the theoretical \(90\%\) design target, is primarily driven by conservative Wald approximations, discrete path overshoot, and the implemented logical exclusion fallback. Because Wald's bounds establish worst-case error guarantees and discrete \(\pm 0.405\) LLR updates routinely cause trajectory overshoots, the test inherently gathers additional confirmatory evidence past the strict threshold. Combined with the single-survivor rescue mechanism, these factors collectively minimize false decisions and elevate overall cryptanalytic recovery performance.
# SideChannelAttack
Differential Power Analysis (DPA) attack on the Advanced Encryption Standard (AES), specifically targeting the inverse S-Box during the decryption process

## 1. Introduction
**Differential Power Analysis (DPA)** is a form of Side-Channel Attack (SCA) that exploits the statistical relationship between the power consumption of a hardware device (like a smart card or microprocessor) and the intermediate data being processed.

While traditional cryptanalysis treats an algorithm as a mathematical "black box," DPA looks at the physical implementation. The provided code specifically implements a **Correlation Power Analysis (CPA)** variant, using the **Hamming Weight** model to correlate guessed power consumption against actual recorded traces to recover the secret AES key byte-by-byte.

## 2. Literature Survey
The field of Power Analysis was pioneered by **Paul Kocher** in 1999, who demonstrated that even subtle variations in power could leak secrets. Key milestones include:
* **Simple Power Analysis (SPA):** Directly observing power traces to identify instructions (e.g., seeing a "square" vs. a "multiply" in RSA).
* **Differential Power Analysis (DPA):** Using statistical means (difference of means) to extract signals from noise.
* **Correlation Power Analysis (CPA):** An evolution of DPA (introduced by Brier et al., 2004) that uses the **Pearson Correlation Coefficient** to match power traces with a power model (like Hamming Weight). This is the specific method utilized in your code.

## 3. Applications
* **Security Auditing:** Hardware manufacturers use these scripts to test the "leakage" of chips before mass production.
* **Forensics:** Extracting keys from locked or protected hardware devices.
* **Smart Card Security:** Testing the resilience of credit cards and SIM cards against physical tampering.
* **Cryptographic Research:** Developing and validating counter-measures like "masking" or "shuffling."

## 4. Algorithm: Correlation Power Analysis (CPA)
The algorithm follows a structured statistical process:
1.  **Data Collection:** Capture power traces ($iTraces$) and corresponding ciphertexts ($iCiphertext$).
2.  **Power Modeling:** Predict the power consumption for every possible key guess ($0-255$) using the Hamming Weight of the S-Box output.
3.  **Statistical Correlation:** Use the Pearson correlation to compare the predictions against the actual power measurements across all time samples.
4.  **Key Extraction:** The guess with the highest absolute correlation spike is identified as the most likely secret key.

## 5. Pseudocode
```text
FUNCTION perform_dpa(Ciphertexts, Traces):
    Initialize RecoveredKeys[16]
    Precompute HammingWeightTable for 0-255
    
    FOR each ByteIndex from 0 to 15:
        FOR each KeyGuess from 0 to 255:
            FOR each TraceIndex:
                Value = Ciphertexts[TraceIndex, ByteIndex] XOR KeyGuess
                IntermediateState = InverseSBox(Value)
                Hypothesis[TraceIndex, KeyGuess] = HammingWeight(IntermediateState)
        
        # Center the data
        Center(Hypothesis)
        Center(Traces)
        
        # Calculate Correlation
        CorrelationMatrix = MatrixMultiply(Hypothesis.Transpose, Traces)
        
        # Find the maximum correlation point
        BestGuess = IndexOfMax(Absolute(CorrelationMatrix))
        RecoveredKeys[ByteIndex] = BestGuess
        
    RETURN RecoveredKeys

## 6. Explanation of the Provided Code


### A. Constants and Tables
* `inv_s`: The Inverse S-Box is used because the attack targets the **decryption** phase or the last round of AES where the ciphertext is XORed with the key and passed through the inverse substitution layer.
* `hamming_weight_8bit_table`: A precomputed list of how many "1" bits are in any byte. This is a common proxy for power consumption: a chip uses more energy to set a bit to "1" than to "0".

### B. The `perform_dpa` Function
* **Centering:** The code subtracts the mean from the traces and hypotheses. This simplifies the Pearson Correlation formula to a basic matrix multiplication, significantly speeding up the calculation.
* **Hypothesis Matrix:** It generates $256$ predictions for every single trace. 
* **Matrix Multiplication:** `np.dot(hyp_hw.T, mean_traces)` is the "engine." It calculates the correlation between all 256 guesses and all time samples simultaneously.
* **Visualization:** It uses `matplotlib` to plot the correlation. In a successful attack, the "correct" key guess will show a distinct, high spike compared to the noise of the incorrect guesses.

## 7. Conclusion
The implementation demonstrates how side-channel leakage can bypass the mathematical strength of AES. Even if the AES algorithm is "unbreakable" by brute force, the physical implementation leaks enough information through power consumption to recover a 128-bit key in seconds using only a few hundred traces. The efficiency of this code relies on **vectorized NumPy operations**, making it capable of handling large datasets.

## 8. Extended and Future Research
* **Multi-Variate Attacks:** Attacking multiple points in time simultaneously to bypass certain protections.
* **Deep Learning SCA:** Using Convolutional Neural Networks (CNNs) to recover keys from traces that are "desynchronized" (where the spikes don't line up perfectly).
* **Countermeasures:**
    * **Masking:** Adding random noise to the data so the Hamming Weight is no longer predictable.
    * **Hiding:** Inserting "dummy" cycles or random delays to jitter the traces in time.
* **Targeting Other Layers:** Extending the attack to the `MixColumns` or `AddRoundKey` layers for different AES implementations.

## 9. Results
<img width="1000" height="400" alt="image" src="https://github.com/user-attachments/assets/dd221867-0d3f-4cd7-8490-be4601e08770" />

This figure is the visual output of the **Correlation Power Analysis (CPA)** step within the DPA algorithm. It shows how well your mathematical "guesses" about the key match the physical reality of the power consumption.

 The X and Y Axes
* **X-axis (Time Samples):** Represents the specific moments in time during the AES operation. Since you used `num_test_samples = 100`, the plot spans from 0 to 100.
* **Y-axis (Correlation):** Represents the statistical correlation between your Hamming Weight hypothesis and the power trace. A higher absolute value (positive or negative) indicates a stronger match.

 The Gray "Noise" Lines
The gray background is a "spaghetti plot" representing the correlation traces for all **incorrect key guesses**. 
* Because there are 256 possible values for a single byte, the code calculates 256 correlation lines. 
* For the 255 wrong keys, the power consumption hypothesis is essentially random compared to the actual data, so their correlation stays low and oscillates randomly around zero.

 The Red Line 
The red line represents the **Correct Key Guess** (in this case, `0xeb`). 
* **The Spike:** Notice the sharp downward spike around **Sample 24**. This is the most critical part of the graph.
* At this specific point in time, the hardware was likely processing the S-Box operation. Because your hypothesis for `0xeb` matches what the hardware actually did, the correlation "spikes" significantly above the background noise.

Why is the spike negative?
In power analysis, a negative spike is just as valid as a positive one. Correlation can range from $-1$ to $+1$. A strong negative correlation simply means that as your predicted Hamming Weight increases, the measured power decreases (or vice-versa), often due to how the physical measurement probe is oriented or how the specific CMOS logic transitions.

Summary of Results
In your specific run:
* **The Algorithm's Logic:** The code looked at all 256 lines, found that the line for `0xeb` reached the highest absolute magnitude (approx. **-130** in this scaled result), and concluded that `0xeb` is the secret key byte.
* **Success Metric:** If the red line were buried deep inside the gray mass, the attack would have failed (likely requiring more traces to reduce the noise). Since the red spike is clearly visible, the attack was successful.


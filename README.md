# High-capacity reversible data hiding in encrypted images using dual-stage bit-plane compression and CP-ABE-based recovery authorization

## Overview

This repository provides a MATLAB implementation of a high-capacity reversible data hiding in encrypted images (RDHEI) scheme based on dual-stage bit-plane compression and CP-ABE-based recovery authorization.

The proposed method first transforms the original image into a prediction-error representation and rearranges the absolute prediction-error image to improve compressibility. Then, dual-stage bit-plane compression based on hierarchical block variable-length coding (HBVLC) is applied to vacate embedding room. The processed image and auxiliary information are encrypted using AES-128 in counter mode (AES-CTR), and additional data are embedded into the encrypted image.

To support fine-grained recovery authorization, the 16-byte AES-CTR master key is encapsulated using ciphertext-policy attribute-based encryption (CP-ABE). In this implementation, the RDHEI algorithm is implemented in MATLAB, while the CP-ABE key encapsulation module is implemented in Java. Only users whose attributes satisfy the predefined access policy can recover the AES key and perform authorized image recovery.

The scheme supports complete reversibility: both the embedded data and the original image can be losslessly recovered.


## Algorithm Pipeline

The overall framework consists of four main stages:
Original Image
      ↓
Prediction Error Computation
      ↓
Absolute Error Rearrangement
      ↓
Dual-Stage Bit-Plane Compression
      ↓
AES-128-CTR Image Encryption
      ↓
Auxiliary Information Encryption
      ↓
Data Embedding
      ↓
Marked Encrypted Image
      ↓
Data Extraction
      ↓
CP-ABE-Based AES Key Recovery
      ↓
AES-CTR Decryption
      ↓
Prediction Error Recovery
      ↓
Lossless Image Reconstruction


## Stage 1: Prediction Error Computation

A **GAP (Gradient-Adjusted Predictor)** is used for interior pixels and a **MED (Median Edge Detector)** for edge pixels to compute the prediction error image.

- The first pixel is kept as-is.
- For each pixel, the prediction error `e = original - predicted` is computed.
- The absolute error and its sign (positive/negative) are stored separately.

The sign labels are recorded as a binary string (0 for positive, 1 for negative) and embedded as part of the auxiliary information.

---

## Stage 2: Hierarchical Block Variable-Length Coding (Core Encoding)

This is the key innovation of the paper. It encodes a binary bit-plane using a **three-level hierarchical block decomposition**.

### 2.1 Block Hierarchy

Given a fundamental block size `W1` (e.g., 16×16):

| Level | Block Size | Number of Sub-blocks |
|-------|-----------|---------------------|
| Level 1 | W1 × W1 (e.g., 16×16) | 1 |
| Level 2 | W2 × W2 (e.g., 8×8) | 4 (2×2 grid) |
| Level 3 | W3 × W3 (e.g., 4×4) | 16 (4×4 grid) |

where `W2 = W1/2`, `W3 = W1/4`.

### 2.2 Threshold Computation

For each level, an adaptive threshold `T` is computed based on the block size:

```
T = floor((W² - 2) / log2(W²))
```

The parameter `p = ceil(log2(T))` is the number of bits needed to encode the count of 1's in a Type-II block.

### 2.3 Encoding Types

Each block is classified into one of three types based on `n1` (the number of 1-valued pixels in the block):

#### Type I (n1 = 0)
The block is all zeros. Only a flag bit is needed.

#### Type II (0 < n1 ≤ T)
The block is sparse. The encoding consists of:
1. **n1 count**: `p` bits storing the number of 1's
2. **Position encoding**: differential coding of the positions of all 1's

For position encoding, the positions of 1's are sorted in raster scan order. Each position is encoded as the difference from the previous position, using the minimum number of bits needed for the remaining range:

```
first position:  bits = ceil(log2(n_pixels))
                 diff = position - 1

subsequent:      bits = ceil(log2(n_pixels - prev_position))
                 diff = position - prev_position - 1
```

This adaptive bit allocation ensures no redundant bits are used.

#### Type III (n1 > T)
The block is dense. The raw bitmap is stored directly with `W²` bits.

### 2.4 Encoding Decision Tree

At **Level 1** (W1×W1 block), the encoding uses a 2-bit prefix:

| Prefix | Meaning | Action |
|--------|---------|--------|
| `00` | Level-1 Type I | Block is all zeros |
| `01` | Level-1 Type II | Encode with Type-II structure |
| `10` | Split to Level 2 | Decompose into 4 sub-blocks, each encoded at Level 2 |
| `11` | Split to Level 3 | Decompose into 16 sub-blocks, each encoded at Level 3 |

**Level 2** (W2×W2 sub-block): Uses a 1-bit prefix:
- `0`: Type I (all zeros)
- `1`: Type II (encode with Type-II structure)

The condition for splitting to Level 2 is: **all 4 sub-blocks have n1 ≤ T2**.

**Level 3** (W3×W3 sub-block): Uses a variable-length prefix:
- `0`: Type I (all zeros)
- `10`: Type II (encode with Type-II structure)
- `11`: Type III (raw bitmap, W3² bits)

### 2.5 Encoding Example

For a 16×16 block (W1=16):
- Level 1 threshold T1: `floor((256-2)/log2(256)) = floor(254/8) = 31`
- Level 2 threshold T2: `floor((64-2)/log2(64)) = floor(62/6) = 10`
- Level 3 threshold T3: `floor((16-2)/log2(16)) = floor(14/4) = 3`

If the block has 5 ones and they are clustered, the encoding might be:
```
01 [n1_count_bits] [position_diff_bits]
```

If the block has 50 ones (n1 > T1), it splits to Level 2, and each 8×8 sub-block is encoded independently.

---

## Stage 3: Bit-plane Processing per Block

For each M×N image divided into non-overlapping blocks of size `t1×t2`:

1. **Identify same MSB planes**: Count consecutive all-zero bit-planes starting from the MSB. These are `same_MSB` planes that can be directly vacated for embedding.

2. **Compress remaining planes**: For each non-zero plane, attempt compression using the hierarchical block coding:
   - If `compressed_size < t1×t2`, the plane is **compressible** (label = 1), and the compressed code is stored.
   - Otherwise, the plane is **not compressible** (label = 0).

3. **Compute N_eb** (number of embeddable planes): `N_eb = same_MSB + count(compressible_planes)`

4. **Rearrange bit-planes**: Move compressible planes above non-compressible ones (`block_plane_move`), so that embeddable planes occupy the upper positions.

---

## Stage 4: Block Sorting & Huffman Coding

Blocks are sorted by `N_eb` in descending order. The same_MSB values are Huffman-coded using a fixed codebook (sorted by frequency for optimal compression).

---

## Stage 5: Encryption

TAfter obtaining the rearranged absolute prediction-error image, the image content is encrypted using AES-128 in counter mode.

A 16-byte AES master key is used:ke ∈ {0,1}^128


## Stage 6: Auxiliary Information Construction

The auxiliary information contains all data needed for recovery:

| Field | Description |
|-------|-------------|
| `same_MSB_Num` | Huffman frequencies (9 values × NumbinLen bits each) |
| `same_MSB_bin` | Huffman-coded same_MSB values for each block |
| `LSB_label_Array` | Compressibility labels for each plane |
| `minority_value_bin` | Minority values for each plane (always 1) |
| `code_length_bin` | Length of each compressed code (8 bits each) |
| `compressed_codes` | Concatenated hierarchical block codes |
| `error_sign_label` | Sign labels for prediction errors |
| `first_pixel` | Original first pixel value (8 bits) |

The auxiliary info is encrypted and embedded alongside the payload data.

---

## Stage 7: Data Embedding

The auxiliary information and encrypted payload data are concatenated into one bitstream. The bitstream is embedded by replacing the embeddable bit-planes of each block in raster scan order, starting from blocks with the highest `N_eb`.

---

---

## Stage 8: CP-ABE-Based AES Key Encapsulation

To enable recovery authorization, the AES master key k_e is encapsulated using ciphertext-policy attribute-based encryption.
The CP-ABE module is implemented in Java and is used to protect the 16-byte AES-CTR key. The MATLAB RDHEI pipeline interacts with the Java CP-ABE module through wrapper functions or command-line calls.
The content owner encrypts k_e under an access policy, for example:("Researcher" AND "Authorized") OR "DataOwner".
Only users whose attributes satisfy the access policy can decrypt the CP-ABE ciphertext and recover k_e.

---


## Stage 9: Data Extraction

1. Read the first two planes of the first block to extract the header (`same_MSB_LSB_Num` and `remain_auxInfo_size`).
2. If needed, read additional header planes from the first block.
3. Extract all embedded bits from each block's embeddable planes.
4. Split the extracted bits into auxiliary information and payload data.
5. Decrypt the auxiliary info and payload data using the respective keys.

---

## Stage 10: Image Recovery

1. Decrypt the marked encrypted image.
2. Parse the auxiliary information to reconstruct:
   - The Huffman-coded `same_MSB` values for each block
   - The `LSB_label` array for each block
   - The compressed hierarchical block codes
3. Decode the compressed bit-planes using the hierarchical block decoder.
4. Recover the original bit-plane order (`block_plane_recover`).
5. Reconstruct the prediction error image.
6. Recover the original image using the inverse predictor (`Recover_error_matrix`), applying the sign labels and the GAP/MED inverse prediction.

---

## Key Advantages

1. **High compression ratio**: The hierarchical block coding adapts to the sparsity of each bit-plane at multiple scales, achieving efficient compression.
2. **Adaptive threshold**: The Type-II threshold `T = floor((W²-2)/log2(W²))` balances between Type-II encoding and raw bitmap storage.
3. **Differential position encoding**: The adaptive bit allocation for position differences minimizes overhead.
4. **Complete reversibility**: Both the embedded data and the original image are losslessly recoverable.
5. **Separable**: Data extraction and image recovery can be performed independently with the respective keys.
6. **Fine-grained recovery authorization**:CP-ABE allows only users satisfying the access policy to recover the AES key.
7. **Hybrid MATLAB-Java design**:MATLAB is used for RDHEI processing, while Java is used for CP-ABE key encapsulation.

---

## File Structure

| File | Description |
|------|-------------|
| `main.m` | Main entry point for the full pipeline |
| `owner.m` | Image owner side: prediction error, compression, encryption |
| `encode.m` | Hierarchical block encoder |
| `decode.m` | Hierarchical block decoder |
| `Pre_error_matrix.m` | GAP/MED prediction error computation |
| `Recover_error_matrix.m` | Inverse prediction for image recovery |
| `block_plane_move.m` | Bit-plane rearrangement for embedding |
| `block_plane_recover.m` | Bit-plane recovery to original order |
| `EncryptionImg.m` | XOR-based image encryption |
| `DecryptionImg.m` | XOR-based image decryption |
| `EncryptionString.m` | Bitstream encryption |
| `DecryptionString.m` | Bitstream decryption |
| `Embed_All_Info.m` | Embed all data into encrypted image |
| `Embed_plane_seq.m` | Embed data into a specific bit-plane |
| `Embed_first_data.m` | Embed initial data |
| `Embed_Aux_Img.m` | Embed auxiliary information |
| `Embed_Aux_Block.m` | Embed auxiliary info into a block |
| `Extract_data.m` | Extract embedded data |
| `Recover_img.m` | Full image recovery |
| `Plane_sequence.m` | Extract a bit-plane from a block |
| `huffman_define.m` | Huffman codebook definition |
| `huffman_encoding.m` | Huffman encoding |
| `Index_to_bm_bn.m` | Index to block coordinate conversion |
| `bin_to_dec.m` / `dec_to_bin.m` | Binary-decimal conversion utilities |
| `psnr.m` / `compute_image_metrics.m` | Image quality metrics |
| `compute_avg_embedding_rate.m` | Average embedding rate calculation |
| `java-cpabe` | Plotting utilities |
| `README.md` |  

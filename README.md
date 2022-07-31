# 项目名称
实现sm4的加密和优化

# 项目内容
1.首先我对sm4算法进行了实现，并且实现了它的ECB模式

2.我对sm4实现了T表查询优化

3.对sm4ECB进行多线程优化

4.对sm4 算法进行SIMD优化


# 项目代码
1.simd优化（单轮）
```c++
void encryption_SIMD(uint32_t* ciphertext, uint32_t* plaintext_for_one_round) {
    __m256i vindex, R0, R1, R2, R3,R_K;
    __m256i index = _mm256_setzero_si256();
    vindex = _mm256_setr_epi32
    (0, 4, 8, 12, 16, 20, 24, 28);
    R0 = _mm256_i32gather_epi32((int*)(plaintext_for_one_round), vindex, 4);
    R1 = _mm256_i32gather_epi32((int*)(plaintext_for_one_round + 1), vindex, 4);
    R2 = _mm256_i32gather_epi32((int*)(plaintext_for_one_round + 2), vindex, 4);
    R3 = _mm256_i32gather_epi32((int*)(plaintext_for_one_round + 3), vindex, 4);
    for (int i = 0; i < 8; ++i)
    {
        R_K = _mm256_i32gather_epi32((int*)
            (rK + 4 * i + 0), index, 4);
        R_K = _mm256_xor_si256(R_K, R1);
        R_K = _mm256_xor_si256(R_K, R2);
        R_K = _mm256_xor_si256(R_K, R3);
        R_K = SIMD_T_func(R_K);
        R0 = _mm256_xor_si256(R0, R_K);
        R_K = _mm256_i32gather_epi32((int*)
            (rK + 4 * i + 1), index, 4);
        R_K = _mm256_xor_si256(R_K, R0);
        R_K = _mm256_xor_si256(R_K, R2);
        R_K = _mm256_xor_si256(R_K, R3);
        R_K = SIMD_T_func(R_K);
        R1 = _mm256_xor_si256(R1, R_K);
        R_K = _mm256_i32gather_epi32((int*)
            (rK + 4 * i + 2), index, 4);
        R_K = _mm256_xor_si256(R_K, R1);
        R_K = _mm256_xor_si256(R_K, R0);
        R_K = _mm256_xor_si256(R_K, R3);
        R_K = SIMD_T_func(R_K);
        R2 = _mm256_xor_si256(R2, R_K);
        R_K = _mm256_i32gather_epi32((int*)
            (rK + 4 * i + 3), index, 4);
        R_K = _mm256_xor_si256(R_K, R2);
        R_K = _mm256_xor_si256(R_K, R1);
        R_K = _mm256_xor_si256(R_K, R0);
        R_K = SIMD_T_func(R_K);
        R3 = _mm256_xor_si256(R3, R_K);
    }
    _mm256_storeu_si256((__m256i*)ciphertext + 0, _mm256_unpackhi_epi64(_mm256_unpacklo_epi32(R3, R2), _mm256_unpacklo_epi32(R1, R0)));
    _mm256_storeu_si256((__m256i*)ciphertext + 1, _mm256_unpackhi_epi64(_mm256_unpacklo_epi32(R3, R2), _mm256_unpacklo_epi32(R1, R0)));
    _mm256_storeu_si256((__m256i*)ciphertext + 2, _mm256_unpackhi_epi64(_mm256_unpacklo_epi32(R3, R2), _mm256_unpacklo_epi32(R1, R0)));
    _mm256_storeu_si256((__m256i*)ciphertext + 3, _mm256_unpackhi_epi64(_mm256_unpacklo_epi32(R3, R2), _mm256_unpacklo_epi32(R1, R0)));
}
```
2.多线程优化（ECB）
```c++
void SM4_ECB_for_thread_main() {
    uint32_t K[36];
    K[0] = MK[0] ^ FK[0]; K[1] = MK[1] ^ FK[1]; K[2] = MK[2] ^ FK[2]; K[3] = MK[3] ^ FK[3];
    for (int i = 4; i < 36; i++) {
        rK[i - 4] = K[i] = K[i - 4] ^ T_for_get_key_func(K[i - 3] ^ K[i - 2] ^ K[i - 1] ^ CK[i - 4]);
    }
    thread t1(SM4_ECB_for_thread, 0);
    thread t2(SM4_ECB_for_thread, 1);
    thread t3(SM4_ECB_for_thread, 2);
    thread t4(SM4_ECB_for_thread, 3);
    thread t5(SM4_ECB_for_thread, 4);
    thread t6(SM4_ECB_for_thread, 5);
    thread t7(SM4_ECB_for_thread, 6);
    thread t8(SM4_ECB_for_thread, 7);
    t1.join();
    t2.join();
    t3.join();
    t4.join();
    t5.join();
    t6.join();
    t7.join();
    t8.join();
}
```
3.普通实现（单轮）
```c++
void encryption(uint32_t* ciphertext,uint32_t* plaintext_for_one_round) {
    uint32_t X[36] = { 0 };
    X[0] = plaintext_for_one_round[0]; X[1] = plaintext_for_one_round[1]; X[2] = plaintext_for_one_round[2]; X[3] = plaintext_for_one_round[3];
    for (int i = 4; i < 36; i++) {
        X[i] = X[i - 4] ^ T_for_roundfunc_func(X[i - 1] ^ X[i - 2] ^ X[i - 3] ^ rK[i-4]);
    }
    ciphertext[0] = X[35]; ciphertext[1] = X[34]; ciphertext[2] = X[33]; ciphertext[3] = X[32];
}

```

3.T表查询优化（单轮）
```c++
void encryption_use_t_box(uint32_t* ciphertext, uint32_t* plaintext_for_one_round) {
    
    uint32_t X[36] = { 0 };
    X[0] = plaintext_for_one_round[0]; X[1] = plaintext_for_one_round[1]; X[2] = plaintext_for_one_round[2]; X[3] = plaintext_for_one_round[3];
    
    for (int i = 4; i < 36; i++) {
        X[i] = X[i - 4] ^ T_for_roundfunc_use_Tbox_func(X[i - 1] ^ X[i - 2] ^ X[i - 3] ^ rK[i - 4]);
    }
    ciphertext[0] = X[35]; ciphertext[1] = X[34]; ciphertext[2] = X[33]; ciphertext[3] = X[32];
}
```

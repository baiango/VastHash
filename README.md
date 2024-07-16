# ✨ VastHash
Previously named `SXAPH-256 - Shift XOR Add Prime Hash 256-Bit`.

VastHash is a non-cryptographic experimental hash function designed for prime-sized HashMap to minimize memory accesses and maximize the performance on AVX2 systems while having distribution comparable to multiplication-based hash.

VastHash is evaluated on passwords for prime-sized HashMap. So, it'll perform much worse on non-prime-sized HashMap and possibly on other data.

It's made from brute-forcing every combination of operators `|, &, <<, >>, ^, and +`. I found claims online, that `| and &` reduces hash distribution and `~` doesn't improve. But, I wasn't able to get consistently reduced hash distribution result in these claims. Still, they're not our best performers on the list. The Rust compiler will replace `-` (vpsubq) with `+` (vpaddq) so that's excluded from the evaluator.

`hash_256_a`, and `hash_256_b` from VastHash are generated with SymPy PRNG (sympy.randprime), then run through the Chi² test with the password 500 times, and filter out the best result. Yet, constants were manually picked due to how the hash function wasn't able to generalize outside its test.

The Chi² test is vulnerable to guided optimization framework like Bayesian optimization from [Optuna](https://optuna.org/), but multiple bucket sizes are used to improve the hash's ability to generalize over different sized array; it's done on all system cores to ensure maximum core utilization.

Previously, VastHash was brute-force optimized with [Optuna](https://optuna.org/), but it was using only half the performance of each core, and, Optuna wasn't more effective than PRNGs on `<<, >>, ^, +` optimizations for its overhead. Now, its constants are brute-forced optimized on SymPy PRNG.

Current constants are found by SymPy, then converted to Rust code, and reverse engineered from Rust compiler's x86 ASM output.

# 🌟 Features
- Hash 65% faster than a SIMD multiply (`mul_hash`)
- Outperforms hash distribution of `djb2`; SHA-256 when on prime-sized array
- Reduced data race (Doesn't rely on the same previous state except when summing)

# ⛈️ Deal-breakers
- Only works with 256-bit SIMD (AVX2)
- Gray-box hash function (Developed through trial and error)
- Extreme bug-prone when implementing as it's sensitive to typos
- Ineffective with non-prime sized array (But more uniform than `mul_hash` and `djb2`)
- Weak diffusion
- Trivial to perform collision attacks
- Non-innovative design (Due to restrictive AVX2 instruction set and faster than 1 multiply requirement)

# 🐍 Original code (Python)
```py
import numpy as np

def vast_hash_impl(input_data, hash_256=np.array([6205865627071447409, 2067898264094941423, 1954899363002243873, 9928278621127670147], dtype=np.uint64)):
	return np.bitwise_xor(input_data, hash_256)

def vast_hash(input_data):
	return np.sum([vast_hash_impl(dat) for dat in input_data])

assert vast_hash_impl(np.array([123, 123, 123, 123], dtype=np.uint64)).all() == np.array([6205865627071447306, 2067898264094941332, 1954899363002243930, 9928278621127670264], dtype=np.uint64).all()
assert vast_hash(np.array([[123, 123, 123, 123], [123, 123, 123, 123]], dtype=np.uint64)) == 3420395603173502432
```

## 🔩 [Rust port](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DEArgoKkl9ZATwDKjdAGFUtEywYgATKUcAZPAZMADl3ACNMYhAANi5SAAdUBUI7Bhc3D28EpJSBAKDQlgio2MtMa1sBIQImYgJ09084q0wbVOragnyQ8Mi9c06GzOaauu7C4okASktUE2Jkdg4AUi8AZjAwZYBWACEqTCYCecwIRLqmMPoAfWSWdCmdgBFljQBBM0wAanN0EBA7n8QCZopJVJJlmtdq8PkofgQgbVkEgQKoABzRa6g/7LADsu2uLBY11oqCY6BMtzwXC8aNIX0JxPJ6GumHieFB9MZtwQJioVBubLwax8DKJ12QADcCMkaWiqaC8S8oTDPvCgWwWJDoe8Yas1jtdgxUITDMB6M89etDYFaIFMM8vgB6J1fABKmApiwUXwMBEYyAAnjD4iYwl8qAwvpKmOZrghYwhrngWPFaBBAqGCNd0EcmCAviCwZIpl8ALSQ7CF0Hgr54nVvV4ATkzJmzuZqde2VaL4P%2BVGIqCZxGITEDEEN0S8Gm2GO2U9xGlxUkkuMkGib9On0VxaKbaK8oI3kibkik63pXCb20ke6ba2iaw0Gi8XkkazRuLW9KbTdpXl3KcuBpXEdw0KRcWeR5dVxF5dV1a09mNU0jAtbY4LeUNw0jaNY2zBMFAQDMGCzHM8wLVZokNXsIXQ0sKzWHtQTrfE9Q3egCC%2BFg2y%2BAiEDrNYnmrYt/gUNMjggDRHhVeCm34Ygvg7L5AmUki2zImoADpCEiCBS3rNim2bPi6y8aFBNwuM%2BOTVN0wAKg7aSG2bJVDIUdxrho5NkFUCA%2BOgxtYLYxtEKNE0WDNNCMKwiMo3c4kvIUUQDGICBAQLGj6MrYSWOc9jME42ooi7XYi21L5aKEyEhJMBgFCYA5cvVf5NX%2BAhRzq7j/TSlMHhY5U8qbIrDQ0Z5NIAd1HeJ2SMa5mQgYa9i4MbJqYabAmAOb0HQBaR0NLwoImqaZs2%2BbFt2A06IC1z4JCy6wpQ80HXQq17tte1HRdL4hFYb4xMwT0vljL4CAQP7ktqaNImSAQQzDWKfg8ry8B8nr7gymsS3LbKi1ywyOKBkcKL2MqoQqx1qsLOqGu%2BesuMwFg2o6hQutOQF9KCmTG3yzjJS4ASaupxq6e5UlyUpWVaV24hNNja54navSgZ9Oy0Dqzjrg5gb8YK6MvAFqn6uF/ExWJMWKSpOUoCK2WFHlxWpll7aIC8KZS2B1WBHMBkte1HXebWA3aqN2mTe5ZlWXZUEID5%2BlJVd/q/dkgnJUkIOhdD3ZTZ5PkBUwSPhS8GPvy%2BDQwmArhwOAqTE7Jr7i/pfb6RL0VLqea6eejbZ05Dprw%2B2gvo8lEvU99rnm2Dmm%2B/FKUZWpWkFUkGPtndn1cZuwKMLh7CowiwIlYM2S6a%2BmMrMTGy00M5sCdbds8yDzHRPEggIBpNYnKvzviEwdzaE4ynT74XPimNMxFSIdiYB/I%2BG5YxKDqKyAAjpsCA39f4WGEn2EAA4hxzRHGOCcewpwzjnAuJcK4vwaGiFuShu59yHnXKeU8XA1gikvNeW8v4HxPhfG%2BNY94NA/j/AeQCXhgIATAvQqCUDuZGSCibE%2BeF4yJk/k2G%2Bak76dgAa0TY1FH4Amfq/dYMwMGSCfn6Qx79LTjw3NfXWqCTB/wNoApRhEJyHlvhpSBScZHNlgZEbMmAkGQHsX/Zukhpx8PnBoNYXAvzbBfO%2BV23iXJyKzl9eKnlMbeVUCogmgIH4iSwYOYco5xyGjiF8Vu9JKrSJcjAhQcCAlBLSkjLJKNfLs3pBkxKENUrs1qdA1Rut8mUxov2YpuDSkEN2NsekVCvi4npGiKCySbH1MaYg5B3S2mo06YjBKWSkpiFqGjB4AyfGdxGRZMZ%2BjzFSVWUZdZ/jNmQG2cWbJpzjFvPBLcXpnzzl1KGZxK5NU9FiXMUWf4ABZN4AANAZjyhoNOeYErZrT3ntM%2BV09FPyjkpX%2BQ8jeSoOAzFoJwbYvBPDcF4KgTgbozDArmAsWm6weCkAIJoElMwADWIBthNk0hEqc0QDxNkoW%2BKhZKOCSEpZy0gtKOC8AUCAARHKOBaBmHAWASA0CpjoJEcglBdXsnoFEBgeBgAIAILQQMfA6D%2BmIMqiAYQ5VhECLUQMnA2VuuYMQQMAB5MI2hWhqrZbqtggh/UMBtXKrAYQTDACcGIWgyrqWkCwBFVCSwtDprwN/Nokof5yswKoVobZs28ECP6KVOa7RhFHH6lwWAvW8HaimFtpBC3EDCEkTATwGaRQ2pymYAomDAAUAANTwJgca/r4iMA7fwQQIgxDsCkDIQQigVDqHVToPQBgjAgFMOYfQeAwjKsgDMVACtUiprLE4L416CBlnoIW2g1V9b3pBrUYABUywHCOCcaqqxdhMElKoLwNKu3EDwFgC9ekygVFSA4BgzhXCND0P4e0Ew%2BhxESMkSoaR0PDGyAR1I4xehRGaOUEN7RRj1GI00RDtGqj0Yo0UXDlh6NDCYwMMY2HKPTFmPMRYQmpUUtIFSnNCqvjHoIMgL45rLXWsDF8CAuBCAkFMjEqYrbh2kD8iAOYBAsyGogMa/VxBgi/U4OiaIZZmLAGQApqQmk2Wek0zBv4cQl3CGSmu6Qvmt1qDlboOIq14gttJeS2Vu75WcH9W2LMj6qCyYZQppTVqbVqZcHq012muC6fZcOnlIB1yaVxE2R8X5YkbjWLiG8%2BhOAytIIzSuAipM0s4EqlVxXd1uyaxwSDkm5UKr0/1mYXaYaeEkEAA)
```rs
#![feature(portable_simd)]
use std::simd::u64x4;
use std::arch::x86_64::{_mm_loadu_si128, _mm_add_epi64, _mm_shuffle_epi32, _mm_cvtsi128_si64};
use std::mem;

#[no_mangle]
#[inline] // Reduces latency
pub fn vast_hash_impl(input_data: u64x4) -> u64x4 {
	input_data ^ u64x4::from_array([6205865627071447409, 2067898264094941423, 1954899363002243873, 9928278621127670147])
}

#[no_mangle]
pub fn vast_hash(input_data: &[u64x4]) -> u64 {
	let mut hash = u64x4::splat(0);
	for dat in input_data.iter() {
		hash += vast_hash_impl(*dat);
	}
	sum_u64x4_icx(hash)
}

#[no_mangle]
pub fn sum_u64x4_scalar(simd: u64x4) -> u64 {
	let arr: [u64; 4] = unsafe { std::mem::transmute(simd) };
	arr[0].wrapping_add(arr[1].wrapping_add(arr[2]).wrapping_add(arr[3]))
}

#[no_mangle]
#[inline] // Same speed as the scalar version
pub fn sum_u64x4_icx(simd: u64x4) -> u64 {
	let arr: [u64; 4] = unsafe { mem::transmute(simd) };
	let v1 = unsafe { _mm_loadu_si128(arr.as_ptr() as *const _) };
	let v2 = unsafe { _mm_loadu_si128((arr.as_ptr().add(2)) as *const _) };
	let v3 = unsafe { _mm_add_epi64(v1, v2) };
	let v4 = unsafe { _mm_shuffle_epi32(v3, 0b11101110) }; // (v3, [2, 3, 2, 3])
	let v5 = unsafe { _mm_add_epi64(v3, v4) };
	unsafe { _mm_cvtsi128_si64(v5) as u64 }
}

pub fn main() {
	{ // vast_hash_impl
		let input_data = u64x4::splat(123);
		let result = vast_hash_impl(input_data);
		assert_eq!(result, u64x4::from_array([6205865627071447306, 2067898264094941332, 1954899363002243930, 9928278621127670264]));
	}{ // vast_hash
		let input_data = vec![u64x4::splat(123), u64x4::splat(123)];
		let result = vast_hash(&input_data);
		assert_eq!(result, 3420395603173502432);
	}{ // sum_u64x4_icx
		let simd = u64x4::from_array([1, 2, 3, 4]);
		assert_eq!(sum_u64x4_icx(simd), sum_u64x4_scalar(simd));

		let simd = u64x4::from_array([5, 6, 7, 8]);
		assert_eq!(sum_u64x4_icx(simd), sum_u64x4_scalar(simd));

		let simd = u64x4::splat(0);
		assert_eq!(sum_u64x4_icx(simd), sum_u64x4_scalar(simd));

		let simd = u64x4::splat(u64::MAX);
		assert_eq!(sum_u64x4_icx(simd), sum_u64x4_scalar(simd));
	}
}
```
## 🍪 ASM output from Rust port
```asm
.LCPI0_0:
		.quad   6205865627071447409
		.quad   2067898264094941423
		.quad   1954899363002243873
		.quad   -8518465452581881469
vast_hash_impl:
		mov     rax, rdi
		vmovaps ymm0, ymmword ptr [rsi]
		vxorps  ymm0, ymm0, ymmword ptr [rip + .LCPI0_0]
		vmovaps ymmword ptr [rdi], ymm0
		vzeroupper
		ret

.LCPI1_0:
		.quad   6205865627071447409
		.quad   2067898264094941423
		.quad   1954899363002243873
		.quad   -8518465452581881469
vast_hash:
		test    rsi, rsi ; if input_data.len() == 0
		je      .LBB1_1 ; Return u64x4::splat(0)
		shl     rsi, 5 ; input_data.len() * 32
		vpxor   xmm0, xmm0, xmm0 ; hash = u64x4::splat(0)
		xor     eax, eax ; `dat` in input_data
		vmovdqa ymm1, ymmword ptr [rip + .LCPI1_0] ; XOR constants
.LBB1_4:
		vpxor   ymm2, ymm1, ymmword ptr [rdi + rax] ; Position of input_data
		vpaddq  ymm0, ymm2, ymm0 ; Add to `hash`
		add     rax, 32 ; Current index
		cmp     rsi, rax ; Check if finished
		jne     .LBB1_4 ; Loop back
		jmp     .LBB1_2 ; Return block
.LBB1_1:
		vpxor   xmm0, xmm0, xmm0 ; hash = u64x4::splat(0)
.LBB1_2: ; `sum_u64x4_icx()` inlined
		vextracti128    xmm1, ymm0, 1
		vpaddq  xmm0, xmm1, xmm0
		vpshufd xmm1, xmm0, 238
		vpaddq  xmm0, xmm0, xmm1
		vmovq   rax, xmm0
		vzeroupper
		ret

sum_u64x4_scalar:
		mov     rax, qword ptr [rdi + 8]
		add     rax, qword ptr [rdi + 16]
		add     rax, qword ptr [rdi + 24]
		add     rax, qword ptr [rdi]
		ret

sum_u64x4_icx:
		vmovdqa xmm0, xmmword ptr [rdi + 16]
		vpaddq  xmm0, xmm0, xmmword ptr [rdi]
		vpshufd xmm1, xmm0, 238
		vpaddq  xmm0, xmm0, xmm1
		vmovq   rax, xmm0
		ret

example::main::he27277a11553942e:
		mov     rax, qword ptr [rip + __rust_no_alloc_shim_is_unstable@GOTPCREL]
		movzx   eax, byte ptr [rax]
		ret
```

# ⏩🚀 Hashing Performance
This is tested with i5-9300H CPU. The speed will be different from different CPUs.
## 🥟 8-bit (small) hash
| CPU      | Bench name       | Time (ps/ns/µs/ms/s) |
|----------|------------------|---------------------|
| i5-9300H | mul_hash_impl_8  | 1.3460 ns           |
|          | djb2_hash_8      | 2.1672 ns           |
|          | vast_hash_impl_8 | 1.2216 ns           |
|          | fnv_1a_hash_8    | 2.1814 ns           |

## 🍔 256-bit (standard) hash
| CPU      | Bench name         | Time (ps/ns/µs/ms/s) |
|----------|--------------------|----------------------|
| i5-9300H | mul_hash_impl_256  | 1.4054 ns            |
|          | djb2_hash_256      | 7.1315 ns            |
|          | vast_hash_impl_256 | 1.2279 ns            |
|          | fnv_1a_hash_256    | 7.1302 ns            |

## 🍉 1 MiB (MP3) hash
| CPU      | Bench name       | Time (ps/ns/µs/ms/s) |
|----------|------------------|----------------------|
| i5-9300H | mul_hash_1mib    | 44.530 µs            |
|          | djb2_hash_1mib   | 587.82 µs            |
|          | vast_hash_1mib   | 23.368 µs            |
|          | fnv_1a_hash_1mib | 1.0407 ms            |

## 📄 Miscellaneous
| CPU      | Bench name       | Time (ps/ns/µs/ms/s) |
|----------|------------------|----------------------|
| i5-9300H | sum_u64x4_scalar | 247.53 ps            |
|          | sum_u64x4_icx    | 247.68 ps            |

Running command:
```py
cargo bench
```

# 🚄🔥 Chi² benchmark (Lower gives better distribution)
`test_hash_distribution()` tests the distribution of hash values generated by a given function for a set of input data. It calculates the chi-squared statistic to measure the deviation of the observed hash distribution from the expected uniform distribution.
```py
const_u64x4_chunks = load_test('BruteX_password.list')

def test_hash_distribution(func, bucket_sizes=[11, 13, 17, 19], u64x4_chunks=const_u64x4_chunks):
	chi2 = 0
	for bucket_size in bucket_sizes:
		buckets = np.zeros(bucket_size, dtype=np.uint64)
		num_hashes = len(u64x4_chunks)

		for i, chk in enumerate(u64x4_chunks):
			hash_value = np.sum([func(c) for c in chk])
			bucket_indices = int(hash_value) % bucket_size
			buckets[bucket_indices] += 1

		expected_frequency = num_hashes // bucket_size
		chi2 += np.sum((buckets - expected_frequency) ** 2)
	return int(chi2)
```
### Hash alignment implementations
```py
def sha256_hash(input_bytes):
	# Compute SHA-256 hash
	hash_object = hashlib.sha256(input_bytes)
	hash_hex = hash_object.hexdigest()

	# Convert the hash value to a 4-element array of 64-bit integers
	hash_int = int(hash_hex, 16)
	hash_array = np.array([(hash_int >> (64 * i)) & 0xffffffffffffffff for i in range(4)], dtype=np.uint64)

	# Return the hash value as a 4-element array of 64-bit integers
	return hash_array

def crc32_hash(input_bytes):
	# Compute CRC32 hash
	hash_object = zlib.crc32(input_bytes)

	# The CRC32 hash value is a single 32-bit integer, so we return it as a 1-element array
	hash_array = np.array([hash_object], dtype=np.uint32)

	# Return the hash value as a 1-element array of 32-bit integers
	return hash_array

def fnv1a_32_hash(input_bytes):
	hash_array = np.array([fnv1a_32(bytes(input_bytes))], dtype=np.uint32)
	return hash_array

def djb2_hash(input_bytes):
	hash_value = np.uint32(5381)
	for b in input_bytes:
		hash_value = hash_value * np.uint32(33) + b
	return hash_value

def mul_hash(input_data, hash_256=np.array([140, 91, 171, 62], dtype=np.uint64)):
	return hash_256 * input_data
```

## 🧮 Prime benchmark
ℹ️ Disclosure: `vast_hash` were evaluated on the [BruteX_password.list](https://weakpass.com/wordlist/1902) and `[11, 13, 17, 19]` bucket sizes.  

### Bucket sizes = `[11, 13, 17, 19]`  
Hash file = [conficker.txt](https://weakpass.com/wordlist/60)  

| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 1,237             |
| djb2_hash       | 8,207             |
| vast_hash       | 779               |
| fnv1a_32_hash   | 683               |
| crc32_hash      | 695               |
| sha256_hash     | 793               |

Running command:
```py
chi2_benchmark(bucket_sizes=[11, 13, 17, 19], hash_path='conficker.txt')
```

### Bucket sizes = `[2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]`  
Hash file = [BruteX_password.list](https://weakpass.com/wordlist/1902)  

| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 20,222,418        |
| djb2_hash       | 8,870,738         |
| vast_hash       | 70,326            |
| fnv1a_32_hash   | 86,620            |
| crc32_hash      | 93,790            |
| sha256_hash     | 79,962            |

```py
chi2_benchmark(bucket_sizes=[i for i in range(1, 101) if is_prime(i)], hash_path='BruteX_password.list')
```

### Bucket sizes = `[2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]`  
Hash file = [rockyou-65.txt](https://weakpass.com/wordlist/87)  

| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 986,582,602       |
| djb2_hash       | 349,747,618       |
| vast_hash       | 732,138           |
| fnv1a_32_hash   | 923,010           |
| crc32_hash      | 739,366           |
| sha256_hash     | 771,564           |

Running command:
```py
chi2_benchmark(bucket_sizes=[i for i in range(1, 101) if is_prime(i)], hash_path="rockyou-65.txt")
```

## 1️⃣4️⃣6️⃣8️⃣9️⃣🔟 Non-prime benchmark
### Bucket sizes = `range(1, 101)`  
Hash file = [BruteX_password.list](https://weakpass.com/wordlist/1902)  
| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 144,201,418       |
| djb2_hash       | 57,182,530        |
| vast_hash       | 1,956,106         |
| fnv1a_32_hash   | 377,188           |
| crc32_hash      | 399,244           |
| sha256_hash     | 316,864           |

Running command:
```py
chi2_benchmark(bucket_sizes=range(1, 101), hash_path='BruteX_password.list')
```

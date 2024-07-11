# ✨ SXAPH-256 - Shift XOR Add Prime Hash 256-Bit
This is an experimental hash function drop-in replacement made from brute-forcing every combination of operators `~, |, &, <<, >>, ^, +, and -`. I found out, `| and &` reduces hash distribution and `~` doesn't improve. SXAPH-256 is designed and trained for prime-sized HashMap. So, it'll perform worse on non-prime-sized HashMap.

# 🌟 Features
- Hash 1.66x faster than with one SIMD multiply (`mul_hash`); it uses (1 Left shift, 1 right shift, 2 XOR, 1 Add)
- Outperform hash distribution of SHA-256 with prime-sized array

# ⛈️ Deal-breakers
- Only works with 256-bit SIMD (AVX2)
- Uninterpretable output

# Original code (Python)
```py
def shift_xor_add_impl(input_data, l=31, r=24, hash_256_a=np.array([10449833687391262720, 12708638996869552128, 12083284059032971264, 5098133568216696832], dtype=np.uint64), hash_256_b=np.array([9858113524293627904, 2849775663957600256, 12247827806936932352, 1651210329918801920], dtype=np.uint64)):
	ls_dat = np.left_shift(input_data, l)
	hash_256_a = np.bitwise_xor(ls_dat, hash_256_a)
	rs_dat = np.right_shift(input_data, r)
	hash_256_b = np.bitwise_xor(rs_dat, hash_256_b)
	return np.add(hash_256_a, hash_256_b)
```

## [Rust port](https://godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:rust,selection:(endColumn:1,endLineNumber:18,positionColumn:1,positionLineNumber:18,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:'%23!!%5Bfeature(portable_simd)%5D%0Ause+std::simd::u64x4%3B%0A%0A%23%5Bno_mangle%5D%0Apub+fn+shift_xor_add_impl(input_data:+u64x4)+-%3E+u64x4+%7B%0A++++let+ls_dat+%3D+input_data+%3C%3C+u64x4::splat(31)%3B%0A++++let+hash_256_a+%3D+ls_dat+%5E+u64x4::from_array(%5B10449833687391262720,+12708638996869552128,+12083284059032971264,+5098133568216696832%5D)%3B%0A++++let+rs_dat+%3D+input_data+%3E%3E+u64x4::splat(24)%3B%0A++++let+hash_256_b+%3D+rs_dat+%5E+u64x4::from_array(%5B9858113524293627904,+2849775663957600256,+12247827806936932352,+1651210329918801920%5D)%3B%0A++++hash_256_a+%2B+hash_256_b%0A%7D%0A%0Afn+main()+%7B%0A++++let+input_data+%3D+u64x4::splat(123)%3B%0A++++let+result+%3D+shift_xor_add_impl(input_data)%3B%0A++++assert_eq!!(result,+u64x4::from_array(%5B1861203148712757248,+15558414740284047360,+5884367880307181568,+6749343681239650304%5D))%3B%0A%7D%0A'),l:'5',n:'1',o:'Rust+source+%231',t:'0')),k:33.333333333333336,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:nightly,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'0',intel:'0',libraryCode:'1',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:rust,libs:!(),options:'-C+opt-level%3D2+-C+target-feature%3D%2Bavx2,%2Bfma',overrides:!(),selection:(endColumn:12,endLineNumber:21,positionColumn:1,positionLineNumber:1,selectionStartColumn:12,selectionStartLineNumber:21,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+rustc+nightly+(Editor+%231)',t:'0')),k:33.31475057813016,l:'4',m:100,n:'0',o:'',s:0,t:'0'),(g:!((h:output,i:(compilerName:'x86-64+gcc+14.1',editorid:1,fontScale:14,fontUsePx:'0',j:1,wrap:'1'),l:'5',n:'0',o:'Output+of+rustc+nightly+(Compiler+%231)',t:'0')),k:33.351916088536505,l:'4',m:100,n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4)
```rs
#![feature(portable_simd)]
use std::simd::u64x4;

#[no_mangle]
pub fn shift_xor_add_impl(input_data: u64x4) -> u64x4 {
	let ls_dat = input_data << u64x4::splat(31);
	let hash_256_a = ls_dat ^ u64x4::from_array([10449833687391262720, 12708638996869552128, 12083284059032971264, 5098133568216696832]);
	let rs_dat = input_data >> u64x4::splat(24);
	let hash_256_b = rs_dat ^ u64x4::from_array([9858113524293627904, 2849775663957600256, 12247827806936932352, 1651210329918801920]);
	hash_256_a + hash_256_b
}

fn main() {
	let input_data = u64x4::splat(123);
	let result = shift_xor_add_impl(input_data);
	assert_eq!(result, u64x4::from_array([1861203148712757248, 15558414740284047360, 5884367880307181568, 6749343681239650304]));
}
```
## ASM output from Rust port
```asm
.LCPI0_0:
		.quad   -7996910386318288896
		.quad   -5738105076839999488
		.quad   -6363460014676580352
		.quad   5098133568216696832
.LCPI0_1:
		.quad   -8588630549415923712
		.quad   2849775663957600256
		.quad   -6198916266772619264
		.quad   1651210329918801920
shift_xor_add_impl:
		vmovdqa ymm0, ymmword ptr [rsi]
		vpsllq  ymm1, ymm0, 31
		vpxor   ymm1, ymm1, ymmword ptr [rip + .LCPI0_0]
		vpsrlq  ymm0, ymm0, 24
		vpxor   ymm0, ymm0, ymmword ptr [rip + .LCPI0_1]
		mov     rax, rdi
		vpaddq  ymm0, ymm1, ymm0
		vmovdqa ymmword ptr [rdi], ymm0
		vzeroupper
		ret
```

# 🚄🔥 Chi² benchmark (Lower gives better distribution)
`test_hash_distribution()` tests the distribution of hash values generated by a given function for a set of input data. It calculates the chi-squared statistic to measure the deviation of the observed hash distribution from the expected uniform distribution.
```py
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
	return chi2
```
### Hash alignment implementations
```py
def sha256_hash(input_data):
	# Convert input data to bytes
	input_bytes = input_data.tobytes()

	# Compute SHA-256 hash
	hash_object = hashlib.sha256(input_bytes)
	hash_hex = hash_object.hexdigest()

	# Convert the hash value to a 4-element array of 64-bit integers
	hash_int = int(hash_hex, 16)
	hash_array = np.array([(hash_int >> (64 * i)) & 0xffffffffffffffff for i in range(4)], dtype=np.uint64)

	# Return the hash value as a 4-element array of 64-bit integers
	return hash_array

def crc32_hash(input_data):
	# Convert input data to bytes
	input_bytes = input_data.tobytes()

	# Compute CRC32 hash
	hash_object = zlib.crc32(input_bytes)

	# The CRC32 hash value is a single 32-bit integer, so we return it as a 1-element array
	hash_array = np.array([hash_object], dtype=np.uint32)

	# Return the hash value as a 1-element array of 32-bit integers
	return hash_array

def mul_hash(input_data, hash_256=np.array([140, 91, 171, 62], dtype=np.uint64)):
	return hash_256 * input_data
```

## 🧮 Prime benchmark
### Bucket sizes = `[11, 13, 17, 19]`  
Hash file = [conficker.txt](https://weakpass.com/wordlist/60)  

| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 1237              |
| shift_xor_add_impl | 223            |
| crc32_hash      | 695               |
| sha256_hash     | 793               |

Running command:
```py
print("test_hash_distribution(mul_hash): ", test_hash_distribution(mul_hash)) # 1237
print("test_hash_distribution(shift_xor_add_impl): ", test_hash_distribution(shift_xor_add_impl)) # 223
print("test_hash_distribution(crc32_hash):" , test_hash_distribution(crc32_hash)) # 695
print("test_hash_distribution(sha256_hash):" , test_hash_distribution(sha256_hash)) # 793
```

### Bucket sizes = `[2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]`  
Hash file = [BruteX_password.list](https://weakpass.com/wordlist/1902)  

| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 20222418          |
| shift_xor_add_hash | 78718          |
| crc32_hash      | 93790             |
| sha256_hash     | 79962             |

Running command:
```py
chi2_benchmark()
```

## 1️⃣4️⃣6️⃣8️⃣9️⃣🔟 Non-prime benchmark
### Bucket sizes = `range(1, 101)`  
Hash file = [BruteX_password.list](https://weakpass.com/wordlist/1902)  
| Hash name       | Non-uniform score |
|-----------------|-------------------|
| mul_hash        | 144201418         |
| shift_xor_add_hash | 1815780        |
| crc32_hash      | 399244            |
| sha256_hash     | 316864            |

Running command:
```py
chi2_benchmark(range(1, 101))
```

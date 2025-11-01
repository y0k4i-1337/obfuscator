# Instructions

## Compiling for Visual Studio 2022

### LegacyPass

Open terminal view in Visual Studio 2022 and run the following commands:

```
wget https://heroims.github.io/obfuscator/LegacyPass/ollvm14.patch -O ollvm14.patch
git clone -b release/14.x https://github.com/llvm/llvm-project.git
cd llvm-project
git apply ../ollvm14.patch
```

After applying the patch, you can proceed to build LLVM:

```
 cmake -G "Visual Studio 17 2022" -S llvm -B build -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld;compiler-rt" -DLLVM_TARGETS_TO_BUILD="X86" -DCMAKE_INSTALL_PREFIX="C:/llvm/14" -DCLANG_ENABLE_BOOTSTRAP=On -DCLANG_BOOTSTRAP_PASSTHROUGH="CMAKE_INSTALL_PREFIX;CMAKE_VERBOSE_MAKEFILE" -DLLVM_ENABLE_NEW_PASS_MANAGER=OFF -DCMAKE_BUILD_TYPE=Release

cmake --build build --config Release --target ALL_BUILD -- /m
```

Use `-v` for more verbose output.

Then, install:

```
cmake --build build --config Release --target INSTALL
```

Note: if using default directories, the above command will probably require administrator privileges.

## How to use it

The simplest way to use Obfuscator-LLVM, is to pass a flag to the LLVM backend from Clang. The current available flags are:

    1. `-fla`: Enable control flow flattening.
    2. `-split`: Enable basic block splitting. Improves the effect of `-fla`.
    3. `-bcf`: Enable bogus control flow.
    4. `-sub`: Enable instruction substitution.

Additional options:

    1. `-aesSeed=<value>`: Set the seed for the random generator used by the obfuscation passes. Default is "".

Imagine that you have a code file named test.c and that you want to use the substitution pass; just call clang like that:

```
$ path_to_the/build/bin/clang test.c -o test -mllvm -sub
```

You can also combine several passes:

```
$ path_to_the/build/bin/clang test.c -o test -mllvm -fla -mllvm -bcf
```

Each pass can also be configured with additional parameters, as follows:

1. Bogus Control Flow (`-bcf`):

    - `-bcf_prob=<value>`: Set the probability of basic blocks to be obfuscated (default is 30, range is 0-100).
    - `-bcf_loop=<value>`: Choose how many times the `-bcf` pass loops on a function (default is 1).

2. Split Basic Blocks (`-split`):

    - `-split_num=<value>`: Set the number of splits per basic block (default is 1).

3. Instruction Substitution (`-sub`):

    - `-sub_loop=<value>`: Choose how many times the `-sub` pass loops on a function (default is 1).


### Example

```
clang -D__CUDACC__ -D_ALLOW_COMPILER_AND_STL_VERSION_MISMATCH -o test -mllvm -split -mllvm -split_num=10 -mllvm -bcf -mllvm -bcf_prob=5 -mllvm -sub -mllvm -aesSeed=DEADBEEF1234432112344321ABCDDDDD test.c
```


## Configuring Visual Studio

To configure Visual Studio to use Obfuscator-LLVM, follow the steps described [here](https://www.bordergate.co.uk/llvm-obfuscation/).

## Troubleshooting

If, during installation, you get errors such as:

```
 CMake Error at projects/compiler-rt/lib/builtins/cmake_install.cmake:39 (file):
    file INSTALL cannot find
    <root directory>/build/$(Configuration)/lib/clang/19/lib/windows/clang_rt.builtins-x86_64.lib":
    No error.
```

Try running the following command in the root directory:

```ps1
Get-ChildItem -Path build -Filter "cmake_install.cmake" -Recurse | ForEach-Object {
    $raw = Get-Content $_.FullName -Encoding Byte
    if ($raw[0] -eq 0xFF -and $raw[1] -eq 0xFE) {
        # UTF-16 LE
        $text = [System.Text.Encoding]::Unicode.GetString($raw)
    } else {
        # UTF-8 fallback
        $text = [System.Text.Encoding]::UTF8.GetString($raw)
    }

    $text = $text -replace [regex]::Escape('$(Configuration)'), 'Release'

    # Save back in same encoding
    if ($raw[0] -eq 0xFF -and $raw[1] -eq 0xFE) {
        [System.IO.File]::WriteAllText($_.FullName, $text, [System.Text.Encoding]::Unicode)
    } else {
        [System.IO.File]::WriteAllText($_.FullName, $text, [System.Text.Encoding]::UTF8)
    }
}
```
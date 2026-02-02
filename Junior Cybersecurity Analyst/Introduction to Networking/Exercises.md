# Subnetting Exercises – by @xcotelo

## 1. Submit the decimal representation of the subnet mask from the following CIDR: 10.200.20.0/27

**CIDR:** `/27`

- A `/27` subnet mask has 27 bits set to 1.
- In the last octet:
  - `11100000` = `224`

**Subnet mask:** 255.255.255.224

---

## 2. Submit the broadcast address of the following CIDR: 10.200.20.0/27

- A `/27` network has blocks of:
  - `256 − 224 = 32` addresses
- The network range is:
  - `10.200.20.0` – `10.200.20.31`

**Broadcast (last address of the range):** 10.200.20.31

---

## 3. Split the network 10.200.20.0/27 into 4 subnets and submit the network address of the 3rd subnet as the answer.

- To split into 4 subnets:
  - `2² = 4` → 2 additional bits are borrowed
  - New prefix: `/29`

- Size of each `/29` subnet:
  - `256 − 248 = 8` addresses per subnet

### Resulting subnets:

| Subnet | Network address |
|------|------------------|
| 1st | `10.200.20.0` |
| 2nd | `10.200.20.8` |
| 3rd | `10.200.20.16` |
| 4th | `10.200.20.24` |

**Network address of the 3rd subnet:** 10.200.20.16

---

## 4. Split the network 10.200.20.0/27 into 4 subnets and submit the broadcast address of the 2nd subnet as the answer.

- 2nd subnet:
  - Network: `10.200.20.8`
  - Size: `8` addresses
  - Range: `10.200.20.8` – `10.200.20.15`

**Broadcast address of the 2nd subnet:** 10.200.20.15

# 5G NR Paging — Complete Interview Guide

> **Author:** Venu | **Date:** February 2026
> Based on hands-on OAI gNB paging implementation (V1 baseline + V2 multi-UE)

---

## Table of Contents

1. [Opening Pitch](#1-opening-pitch)
2. [Complete Detailed Flow — Tell This in One Go](#2-complete-detailed-flow--tell-this-in-one-go)
3. [Architecture Overview](#3-architecture-overview)
4. [Implementation Architecture — IPC, Data Structures & Threading](#4-implementation-architecture--ipc-data-structures--threading)
5. [NGAP Layer](#5-ngap-layer)
6. [RRC Layer](#6-rrc-layer)
7. [MAC Layer](#7-mac-layer)
8. [Wireshark Integration](#8-wireshark-integration)
9. [V2 Key Improvements](#9-v2-key-improvements)
10. [Interview Q&A Bank](#10-interview-qa-bank)
11. [Quick Reference Card](#11-quick-reference-card)

---

## 1. Opening Pitch


> *"I implemented 5G NR SA paging end-to-end in OAI's gNB stack. The flow starts at the NGAP layer receiving paging from the AMF with the UE's 5G-S-TMSI and TAI list. At RRC, I compute Paging Frame and Paging Occasion per TS 38.304 — UE_ID is derived from the S-TMSI, then PF and PO are calculated from the DRX cycle, the ratio N, and Ns. I encode the PCCH message using ASN.1 and pass it directly to MAC, bypassing PDCP/RLC since paging is broadcast with no per-UE context. At MAC, I schedule it on the matching frame/slot using DCI Format 1_0 with P-RNTI on CORESET0. I also modified the Wireshark T-tracer to correctly decode P-RNTI PDUs. In a second iteration, I extended it for multi-UE paging, fixed the DCI bit packing to match TS 38.212 exactly, and handled PHY-layer rate matching crashes for small paging TBs."*



---

## 2. Complete Detailed Flow — Tell This in One Go

Use this when the interviewer says *"Walk me through the complete flow in detail"* or *"Explain the implementation end-to-end"*. This covers **spec + implementation + IPC + data structures + V2** in one continuous narrative (~4–5 minutes).

---

### Why Paging Exists

> *"Paging is how the 5G network wakes up a UE that's in IDLE mode. When the UE goes IDLE to save battery, the gNB releases its context and the AMF only knows the UE's Tracking Area — not which specific cell it's camped on. If downlink data arrives at the UPF, it notifies SMF, SMF notifies AMF, and AMF sends an NGAP Paging message to every gNB in the UE's registered Tracking Area list."*

### NGAP Layer — Receiving the Trigger

> *"At the gNB, the NGAP Paging message arrives over SCTP on the N2 interface. My NGAP handler — `ngap_gNB_handle_paging()` — extracts two key things: the UE's **5G-S-TMSI** (which is 48 bits — AMF Set ID 10 bits, AMF Pointer 6 bits, and 5G-TMSI 32 bits) and the **TAI list** (MCC, MNC, and the 24-bit TAC). Then it needs to forward this to RRC. OAI uses an **ITTI message-passing framework** — NGAP allocates a `MessageDef` struct of type `NGAP_PAGING_IND`, fills in the S-TMSI and TAI, and calls `itti_send_msg_to_task()` to post it asynchronously to RRC's task queue. This is important — it's async, NGAP doesn't block waiting for RRC. Under the hood, each layer runs as a separate pthread with its own ITTI queue backed by a linked list and pthread condition variable."*

### RRC Layer — Computing When to Page

> *"RRC picks up the ITTI message from its event loop. First, in `rrc_gNB_process_PAGING_IND()`, I verify the TAI — does our cell's MCC/MNC/TAC match any entry in the paging message's TAI list? If no match, we discard it. If yes, I call `rrc_gNB_generate_pcch_msg()`, which is where the core logic lives."*

> *"**PF/PO Calculation per TS 38.304 Section 7.1:** UE_ID equals the full 48-bit 5G-S-TMSI mod 1024 — this gives a 10-bit value. Then the Paging Frame is: PF = (T/N) times (UE_ID mod N) plus an offset, all mod T, where T is the DRX cycle (e.g. 128 frames) and N is the number of paging frames per cycle. The Paging Occasion slot is: i_s = floor of (UE_ID/N) mod Ns, then PO = i_s times (slots_per_frame / Ns). This two-level formula distributes UEs across both frames and slots — UE_ID mod N spreads them across frames, and (UE_ID/N) mod Ns spreads them across slots within a frame."*

> *"I store this PF/PO in a **shared global array** — `NR_UE_PF_PO[CC_id][UE_index]` — which is a struct containing the enable flag, UE_ID, PF, PO, and DRX cycle T. This array is protected by a **pthread mutex** because RRC writes it and MAC reads it from different threads."*

### RRC Layer — Encoding and Transfer to MAC

> *"Next, I encode the **PCCH message** using ASN.1 UPER encoding via `do_NR_Paging()` — it creates a PagingRecord with the UE's ng-5G-S-TMSI as a 48-bit BIT_STRING. The encoded output is typically 7–24 bytes."*

> *"Now the transfer to MAC — this is a key implementation detail. I use a **function-pointer callback** interface: `rrc->mac_rrc.dl_rrc_message_transfer()`. I package the encoded PCCH bytes into an `f1ap_dl_rrc_message_t` struct but with a **sentinel `srb_id = 0xFF`**. Normal DL messages use srb_id 1 or 2 for SRB1/SRB2. The 0xFF tells MAC: this is paging, not unicast — store it in the pcch buffer, do **not** forward to RLC. Why bypass PDCP and RLC? Three reasons: no ciphering needed (it's broadcast), no segmentation needed (message is tiny), and no per-UE context exists (UE is IDLE, no SRB is established)."*

> *"Why function pointers instead of ITTI? In monolithic gNB, RRC and MAC are in the same process — direct function call is faster and simpler. In a CU/DU split, this same callback gets replaced by F1AP messages over a network socket — the abstraction makes the swap transparent."*

### MAC Layer — Storing and Scheduling

> *"On the MAC side, `dl_rrc_message_transfer()` detects `srb_id == 0xFF`, calls `nr_build_pcch_mac_pdu()` to add a MAC subheader, stores the result in a **dedicated `pcch_pdu[128]` buffer** inside the MAC instance, and sets `pcch_pending = true`."*

> *"Now the scheduling. The main scheduler — `gNB_dlsch_ulsch_scheduler()` — runs **every slot**, that's every 0.5ms at 30 kHz SCS, so 2000 times per second. Inside it, `schedule_nr_paging()` performs four checks: is `pcch_pending` true? Does the current frame match PF (via `frameP % T == PF_min`)? Does the current slot match PO? Is the Type0-PDCCH common search space available on this frame? 99.99% of the time, one of these fails and it returns immediately. But when all four match, it builds the FAPI messages."*

### MAC → PHY — FAPI Interface

> *"MAC talks to PHY through **FAPI** — two messages per paging transmission. First, **DL_TTI.request** contains two PDUs: a **PDCCH PDU** with DCI Format 1_0 scrambled with P-RNTI (0xFFFE) — specifying aggregation level, CCE index, and the packed DCI bit payload — and a **PDSCH PDU** with the resource allocation: RB start, RB size, MCS 0 (QPSK for coverage), number of symbols, and a `PDU_index`. Second, **TX_Data.request** carries the actual PCCH payload bytes, tagged with the **same PDU_index**."*

> *"**PDU_index synchronization is critical** — PHY matches the scheduling info in DL_TTI.request with the payload in TX_Data.request using this index. If they mismatch, PHY transmits the wrong data on air — for example, SIB1 payload on paging resources. After transmission, the scheduler sets `pcch_pending = false` and clears the buffer."*

### PHY → Air → UE

> *"PHY does channel coding with LDPC, rate matching, scrambling with P-RNTI, QPSK modulation, maps to PRBs, beamforms using the same beam as the SSB the UE monitors, and transmits. The UE, which wakes up to monitor P-RNTI in Type0-PDCCH CSS on CORESET0 at its calculated PF/PO, decodes the DCI, finds the PDSCH allocation, reads the paging record, checks if its own S-TMSI matches, and if so, initiates RRC Connection Resume or Setup."*

### V2 Improvements (Mention Only If Asked)

> *"In V2, I hardened this for correctness and multi-UE support. Four changes: (1) **DCI bit packing** — V1 wrongly reused SI-RNTI field layout; P-RNTI DCI has Short Message Indicator and TB Scaling fields instead of HARQ/NDI/TPC fields since paging has no feedback, per TS 38.212. (2) **UE identity** — V1 used raw 32-bit m_tmsi for UE_ID; the spec requires the full 48-bit 5G-S-TMSI mod 1024 — I had to bit-shift AMF Set ID, AMF Pointer, and 5G-TMSI together. (3) **Multi-UE** — V1 hardcoded PF/PO table to index [0][0]; V2 uses a two-pass insertion: first scan for existing entry to update, then find first free slot. (4) **PHY rate matching crash** — small paging TBs (~7 bytes) caused the LDPC rate matcher's filler offset to exceed the circular buffer size, causing OOB memory access. I added clamping and a repetition loop to handle this edge case. This was a great cross-layer debugging story — a MAC scheduling decision exposed a PHY edge case."*


---

## 3. Architecture Overview

### End-to-End Flow

```
AMF ──NGAP Paging──▶ gNB-CU (RRC) ──F1AP/direct──▶ gNB-DU (MAC) ──FAPI──▶ PHY ──RF──▶ UE
```

### Channel Mapping (3GPP TS 38.300)

| Layer | Channel | Purpose |
|-------|---------|---------|
| **Logical** | PCCH (Paging Control Channel) | Carries paging message |
| **Transport** | PCH (Paging Channel) | Transport for paging |
| **Physical** | PDCCH (DCI 1_0 with P-RNTI) | Tells UE where PDSCH is |
| **Physical** | PDSCH | Carries actual paging payload |

### RNTI Values

| RNTI | Value | Used For |
|------|-------|----------|
| **SI-RNTI** | `0xFFFF` | System Information (SIB) |
| **P-RNTI** | `0xFFFE` | Paging |
| **RA-RNTI** | Computed | Random Access Response |
| **C-RNTI** | Assigned | Connected UE unicast |

### Key Spec References

| Spec | Section | Topic |
|------|---------|-------|
| TS 38.304 | §7.1 | PF/PO calculation formula |
| TS 38.212 | §7.3.1.2.1 | DCI Format 1_0 for P-RNTI fields |
| TS 38.213 | §13 | PDCCH monitoring for paging (Type0-CSS) |
| TS 38.331 | §5.3.14 | PCCH message structure |
| TS 38.321 | §6.1.1 | MAC PDU for PCCH |

---

## 4. Implementation Architecture — IPC, Data Structures & Threading

This section covers **how the code is actually wired together** — the IPC mechanisms, key data structures, threading model, and message flow between layers. These are commonly asked implementation questions.

### 4.1 How Does the AMF Know It Needs to Page?

Before anything reaches the gNB, the **Core Network (CN)** side triggers paging:

1. UE registers with AMF → AMF stores UE's **5G-S-TMSI**, **registered TAI list**, and **DRX parameters**
2. UE transitions to **RRC_IDLE** (or RRC_INACTIVE) → gNB releases its context, tells AMF via `UEContextReleaseComplete`
3. Now AMF only knows the UE's **Tracking Area** — not which specific cell it's in
4. Downlink data/call arrives at UPF → UPF notifies SMF → SMF notifies AMF
5. AMF sends **NGAP Paging** to **every gNB** in the UE's registered TAI list

> [!NOTE]
> The AMF doesn't know the UE's exact cell — that's the whole point of paging. It broadcasts to all gNBs in the TA, and only the UE in the right cell (monitoring the right PF/PO) will respond.

### 4.2 Overall Inter-Layer Communication Map

```
┌─────────┐   SCTP/NGAP    ┌─────────┐   ITTI msg     ┌─────────┐   func ptr    ┌─────────┐    FAPI     ┌─────────┐
│   AMF   │ ──────────────▶ │  NGAP   │ ─────────────▶ │   RRC   │ ────────────▶ │   MAC   │ ──────────▶ │   PHY   │
│ (Core)  │   over N2 intf  │ handler │  MessageDef    │  layer  │ mac_rrc cb   │  sched  │  DL_TTI +  │         │
└─────────┘                 └─────────┘  (async queue)  └─────────┘  (direct)     └─────────┘  TX_Data    └─────────┘

   External                  Thread 1                   Thread 1                  Thread 2      Thread 2→3
   (network)                 (SCTP rx)                  (RRC task)                (MAC task)    (PHY task)
```

### 4.3 IPC Mechanism: NGAP → RRC (ITTI Message Passing)

OAI uses an internal framework called **ITTI (Inter-Task Interface)** for communication between protocol layers running in **different threads**.

**How it works:**
```c
// NGAP handler creates a message:
MessageDef *msg_p = itti_alloc_new_message(TASK_NGAP, 0, NGAP_PAGING_IND);

// Fill in the paging data:
NGAP_PAGING_IND(msg_p).ue_paging_identity.s_tmsi.m_tmsi   = tmsi;
NGAP_PAGING_IND(msg_p).ue_paging_identity.s_tmsi.amf_set_id = amf_set_id;
NGAP_PAGING_IND(msg_p).tai_list[0].tac  = tac;
NGAP_PAGING_IND(msg_p).tai_list[0].plmn = plmn;

// Post to RRC task's queue:
itti_send_msg_to_task(TASK_RRC_GNB, instance, msg_p);
```

**Under the hood — ITTI internals:**

| Concept | Implementation |
|---------|----------------|
| **Message** | `MessageDef` struct — has header (task ID, msg ID, size) + payload union |
| **Queue** | Per-task FIFO queue (backed by a linked list + pthread condition variable) |
| **Sending** | `itti_send_msg_to_task()` → enqueues message, signals the target task's condvar |
| **Receiving** | `itti_receive_msg()` → blocks on condvar until message arrives, dequeues |
| **Threading** | Each protocol layer runs as a separate **pthread** with its own ITTI task loop |

```c
// RRC task main loop (simplified):
void *rrc_gnb_task(void *args) {
    while (1) {
        MessageDef *msg_p;
        itti_receive_msg(TASK_RRC_GNB, &msg_p);  // blocks until message
        switch (ITTI_MSG_ID(msg_p)) {
            case NGAP_PAGING_IND:
                rrc_gNB_process_PAGING_IND(msg_p, instance);
                break;
            // ... other message types
        }
    }
}
```

> [!IMPORTANT]
> **Interview point:** ITTI is **asynchronous** — NGAP posts the message and returns immediately. RRC picks it up when it gets to it in its event loop. This decouples the layers and prevents NGAP from blocking on RRC processing.

### 4.4 IPC Mechanism: RRC → MAC (Function Pointer Callback)

RRC and MAC communicate via a **callback interface** — a struct of function pointers that MAC registers at init time.

```c
// Callback interface definition:
typedef struct nr_mac_rrc_dl_if_s {
    // For unicast DL RRC messages and paging:
    void (*dl_rrc_message_transfer)(sctp_assoc_t assoc_id, const f1ap_dl_rrc_message_t *dl_rrc);
    // ... other callbacks for SIB, reconfiguration, etc.
} nr_mac_rrc_dl_if_t;

// In gNB_rrc_inst_t (RRC instance):
nr_mac_rrc_dl_if_t mac_rrc;  // Function pointers to MAC

// RRC calls MAC like this:
rrc->mac_rrc.dl_rrc_message_transfer(assoc_id, &dl_rrc_msg);
```

**For paging specifically:**
```c
// RRC packages paging as a "fake" DL RRC message:
f1ap_dl_rrc_message_t dl_rrc = {
    .rrc_container        = pcch_encoded_buffer,
    .rrc_container_length = pcch_length,
    .srb_id               = 0xFF   // ← Sentinel: "this is paging, not unicast!"
};
rrc->mac_rrc.dl_rrc_message_transfer(assoc_id, &dl_rrc);

// MAC side checks srb_id:
if (dl_rrc->srb_id == 0xFF) {
    // It's paging! Store in pcch buffer, DON'T send to RLC
    mac->pcch_pdu_size = nr_build_pcch_mac_pdu(...);
    mac->pcch_pending = true;
    return;
}
```

> [!NOTE]
> **Why function pointers instead of ITTI?** In monolithic gNB, RRC and MAC are in the **same process** — function pointers give direct, synchronous calls (faster, simpler). In CU/DU split, this callback would be replaced by **F1AP** messages over a network socket.

### 4.5 IPC Mechanism: MAC → PHY (FAPI Interface)

MAC talks to PHY via the **FAPI (Functional API)** — an NFAPI-based message interface. Two messages are sent per paging transmission:

| Message | Struct | Contains |
|---------|--------|----------|
| **DL_TTI.request** | `nfapi_nr_dl_tti_request_t` | PDCCH PDU (DCI with P-RNTI) + PDSCH PDU (resource allocation) |
| **TX_Data.request** | `nfapi_nr_tx_data_request_t` | Actual PCCH payload bytes, linked by `PDU_index` |

```c
// MAC fills these every slot and passes to PHY:
nfapi_nr_dl_tti_request_t  DL_req;   // "What to schedule on air"
nfapi_nr_tx_data_request_t TX_req;   // "Here's the actual data"

// PHY matches them: DL_req.PDU[i].PDU_index == TX_req.PDU[j].PDU_index
```

> [!IMPORTANT]
> **Critical implementation detail:** PDU_index is how PHY knows which payload belongs to which scheduled PDSCH. If paging's PDU_index = 5, PHY looks for TX_req entry with PDU_index = 5. A mismatch means PHY transmits **wrong data** on air.

### 4.6 Key Data Structures

#### a) Paging PF/PO Table (Shared between RRC & MAC)

```c
typedef struct NR_UE_PF_PO_s {
    bool     enable_flag;       // Is this entry active?
    uint32_t ue_index_value;    // UE_ID = 5G-S-TMSI mod 1024
    uint16_t PF_min;            // Calculated Paging Frame
    uint8_t  PO;                // Calculated Paging Occasion (slot)
    uint16_t T;                 // DRX cycle length
} NR_UE_PF_PO_t;

// Global array — cross-thread shared:
NR_UE_PF_PO_t NR_UE_PF_PO[NFAPI_CC_MAX][MAX_MOBILES_PER_GNB];
pthread_mutex_t nr_ue_pf_po_mutex;   // Guards access
```

**Who writes:** RRC (when paging arrives from AMF)
**Who reads:** MAC scheduler (every slot — 2000×/sec at 30kHz)
**Protection:** `pthread_mutex_t` — RRC locks to write, MAC locks to read

#### b) PCCH PDU Buffer (Inside MAC instance)

```c
// Part of gNB_MAC_INST:
uint8_t pcch_pdu[128];       // Encoded PCCH payload (max 128 bytes)
int     pcch_pdu_size;       // Actual size after encoding
bool    pcch_pending;        // Flag: "scheduler, transmit this!"
int     freeUeIndexPaging;   // V2: which PF/PO entry to use
```

**Lifecycle:**
1. RRC encodes PCCH → stores in `pcch_pdu[]` → sets `pcch_pending = true`
2. MAC scheduler checks `pcch_pending` every slot
3. When PF/PO match → transmits → sets `pcch_pending = false`

#### c) NGAP Paging Message (ITTI Payload)

```c
typedef struct ngap_paging_ind_s {
    struct {
        struct {
            uint16_t amf_set_id;    // AMF Set ID (10b) + AMF Pointer (6b)
            uint32_t m_tmsi;        // 5G-TMSI (32 bits)
        } s_tmsi;
    } ue_paging_identity;

    struct {
        uint16_t mcc, mnc;
        uint32_t tac;               // 24-bit Tracking Area Code
    } tai_list[256];
    int     tai_size;               // Number of TAIs
} ngap_paging_ind_t;
```

#### d) FAPI PDU Structures (MAC → PHY)

```c
// PDCCH PDU — tells PHY how to send DCI:
nfapi_nr_dl_tti_pdcch_pdu_rel15_t {
    uint16_t RNTI;              // 0xFFFE for paging
    uint8_t  AggregationLevel;  // 4 or 8 CCEs typically
    uint16_t CceIndex;
    uint8_t  dci_bits[];        // Packed DCI payload
};

// PDSCH PDU — tells PHY resource allocation:
nfapi_nr_dl_tti_pdsch_pdu_rel15_t {
    uint16_t rnti;              // 0xFFFE
    uint16_t BWPSize, rbStart, rbSize;
    uint8_t  mcsIndex, qamModOrder, nrOfLayers;
    uint32_t TBSize;
    uint16_t PDU_index;         // Links to TX_Data.request
};
```

### 4.7 Threading Model

```
Thread 1: SCTP Receiver     → receives NGAP from AMF, posts ITTI msg
Thread 2: RRC Task           → picks up ITTI msg, computes PF/PO, encodes PCCH,
                               writes to shared PF/PO table & pcch_pdu buffer
Thread 3: MAC Scheduler      → runs every slot (0.5ms), reads PF/PO table,
                               builds FAPI PDUs when match found
Thread 4: PHY L1 Thread      → receives FAPI messages, does channel coding,
                               modulation, and transmits on air
```

**Synchronization points:**

| Shared Resource | Writer | Reader | Protection |
|----------------|--------|--------|------------|
| `NR_UE_PF_PO[][]` | RRC | MAC | `pthread_mutex_t` |
| `pcch_pdu[]` + `pcch_pending` | RRC (via MAC callback) | MAC scheduler | Written atomically before flag set |
| FAPI DL_req / TX_req | MAC | PHY | Slot-synchronized (producer-consumer) |

### 4.8 Complete Message Flow — Data Structure Lifecycle

```
 ① AMF sends NGAP Paging over SCTP
              │
              ▼
 ② ngap_gNB_handle_paging() extracts fields into ngap_paging_ind_t
              │
              ▼
 ③ itti_send_msg_to_task(TASK_RRC_GNB, msg_p)  ← ASYNC, returns immediately
              │
              ▼ (RRC thread picks up from its ITTI queue)
 ④ rrc_gNB_process_PAGING_IND() — TAI match check
              │
              ▼
 ⑤ rrc_gNB_generate_pcch_msg()
    ├── Computes PF/PO from UE_ID = 5G-S-TMSI mod 1024
    ├── Writes NR_UE_PF_PO[cc][idx]  ← mutex locked
    ├── Encodes PCCH via do_NR_Paging() → ASN.1 UPER bytes
    └── Calls rrc->mac_rrc.dl_rrc_message_transfer() ← function pointer
              │
              ▼ (direct call, same process in monolithic)
 ⑥ MAC dl_rrc_message_transfer()
    ├── Detects srb_id == 0xFF → this is paging
    ├── nr_build_pcch_mac_pdu() → adds MAC subheader
    ├── Stores in mac->pcch_pdu[]
    └── Sets mac->pcch_pending = true
              │
              ▼ (MAC scheduler runs every 0.5ms)
 ⑦ schedule_nr_paging() — checks every slot:
    ├── pcch_pending? → yes
    ├── frameP % T == PF_min? → checks current frame
    ├── slotP == PO? → checks current slot
    ├── Match! → builds PDCCH PDU (DCI 1_0 + P-RNTI)
    ├── Builds PDSCH PDU (resource allocation)
    ├── Attaches payload to TX_Data.request (PDU_index linked)
    └── Sets pcch_pending = false
              │
              ▼ (FAPI interface)
 ⑧ PHY receives DL_TTI.request + TX_Data.request
    ├── LDPC encoding → rate matching → scrambling
    ├── QPSK modulation → PRB mapping → beamforming
    └── Transmits on air
              │
              ▼ (over RF)
 ⑨ UE in IDLE monitors P-RNTI on its PF/PO
    ├── Decodes DCI → finds PDSCH allocation
    ├── Reads PagingRecord → checks S-TMSI match
    └── If match → initiates RRC Connection Resume/Setup
```

### 🎤 How to Speak About This

> *"In terms of implementation, OAI uses an ITTI message-passing framework for inter-layer communication. NGAP runs in its own SCTP thread, and when paging arrives, it packages the S-TMSI and TAI into an ITTI message and posts it asynchronously to RRC's task queue. RRC picks it up, computes PF/PO, and calls MAC through a function-pointer callback registered at init time. I use a sentinel srb_id of 0xFF to tell MAC this is paging, not unicast — so MAC stores it in a dedicated pcch_pdu buffer instead of forwarding to RLC. The PF/PO table is a shared global array protected by a pthread mutex since RRC and MAC are different threads. MAC then talks to PHY via FAPI — two messages per paging: DL_TTI.request for scheduling metadata and TX_Data.request for the payload, linked by PDU_index."*

---

## 5. NGAP Layer

### What Happens (`ngap_gNB_handlers.c`)

AMF sends **NGAP Paging** message when it needs to reach an IDLE/INACTIVE UE. The gNB's NGAP handler:

1. **Extracts paging identity**: AMF Set ID (10 bits), AMF Pointer (6 bits), 5G-TMSI (32 bits)
2. **Extracts TAI list**: MCC, MNC, TAC for each Tracking Area
3. **Forwards to RRC**: Calls `rrc_gNB_process_PAGING_IND()`

### Bugs Fixed in the Implementation

**Bug 1: Stream-0 rejection** — Original code rejected paging on SCTP stream 0 with `return -1`. This is wrong because some AMF implementations send paging on stream 0. Fix: allow it through.

**Bug 2: TAC parsing** — Used `OCTET_STRING_TO_INT16` but 5G TAC is 3 bytes (24 bits). Fix: `OCTET_STRING_TO_INT24`.

**Bug 3: TMSI format string** — Used `%d` (signed) for m_tmsi but it's `uint32_t`. Fix: `%u`.

### 🎤 How to Speak About This

> *"At NGAP, the main challenge was correctness — the original code rejected paging on SCTP stream 0, which some AMF implementations use. I also fixed the TAC parsing from 16-bit to 24-bit, because 5G TAC is 3 bytes unlike 4G's 2-byte TAC."*

---

## 6. RRC Layer

### 4.1 NGAP→RRC Handoff (`rrc_gNB_NGAP.c`)

```c
int rrc_gNB_process_PAGING_IND(MessageDef *msg_p, instance_t instance)
```

For each TAI in the paging message:
1. Check if gNB serves this TAI (MCC/MNC/TAC match against configuration)
2. If match → call `rrc_gNB_generate_pcch_msg()` for each CC

**Why verify TAI?** An IDLE UE is tracked at TAI granularity. AMF pages across all gNBs in the UE's registered TAI list. Each gNB only responds if it actually serves that TAI.

### 4.2 PF/PO Calculation (`rrc_gNB.c`) — Per TS 38.304 §7.1

This is the **core algorithm**. Given a UE's TMSI, compute *exactly which frame and slot* the paging should be transmitted in.

**Step A — Get DRX cycle T:**
```c
const unsigned int Ttab[4] = {32, 64, 128, 256};  // rf32..rf256
T = Ttab[Tc];  // e.g., T = 256 frames
```
T comes from `defaultPagingCycle` in SIB1's PCCH-Config.

**Step B — Calculate N (total paging frames per cycle):**

N depends on `nAndPagingFrameOffset` from PCCH-Config:

| Config | N | PF offset |
|--------|---|-----------|
| oneT | T | 0 |
| halfT | T/2 | 1 |
| quarterT | T/4 | 3 |
| oneEighthT | T/8 | 7 |
| oneSixteenthT | T/16 | 15 |

**Step C — Calculate UE_ID:**
```c
UE_ID = TMSI % N;   // V1 (simplified)
// V2 fix: UE_ID = 5G-S-TMSI mod 1024 (per spec)
```

**Step D — Calculate Paging Frame (PF):**
```c
PF = ((T/N) × (UE_ID mod N) + PF_offset) % T
```
This distributes UEs evenly across N frames within a DRX cycle.

**Step E — Get Ns (paging occasions per frame):**

| `ns` config | Ns value | PO slots (20 slots/frame @ 30kHz) |
|-------------|----------|-----------------------------------|
| one | 1 | {0} |
| two | 2 | {0, 10} |
| four | 4 | {0, 5, 10, 15} |

**Step F — Calculate Paging Occasion slot:**
```c
i_s = (UE_ID / N) % Ns;     // Which PO within the PF
PO  = i_s × (20 / Ns);      // Actual slot number
```

**Why this works for load balancing:**
- `UE_ID % N` → distributes UEs across **frames**
- `(UE_ID / N) % Ns` → for UEs in the same frame, distributes across **slots**
- Result: ~uniform distribution across all paging occasions

### 4.3 Storing PF/PO — Shared Data Structure

```c
typedef struct NR_UE_PF_PO_s {
    bool     enable_flag;       // Is this slot in use?
    uint32_t ue_index_value;    // UE_ID
    uint16_t PF_min;            // Paging Frame
    uint8_t  PO;                // Paging Occasion (slot)
    uint16_t T;                 // DRX cycle
} NR_UE_PF_PO_t;

// Global: [CC_id][UE_index]
NR_UE_PF_PO_t NR_UE_PF_PO[NFAPI_CC_MAX][MAX_MOBILES_PER_GNB];
pthread_mutex_t nr_ue_pf_po_mutex;
```

**Why global with mutex?** RRC and MAC run in **different threads**. RRC writes PF/PO when paging arrives; MAC reads it every slot to check for matches. Mutex prevents data races.

### 4.4 PCCH Message Encoding

```c
int length = do_NR_Paging(instance, buffer, tmsi);
```

Inside `do_NR_Paging()`:
- Creates `NR_PCCH_Message_t` (ASN.1)
- Adds `PagingRecord` containing the UE's `ng-5G-S-TMSI`
- Encodes using **UPER** (Unaligned Packed Encoding Rules)
- Returns byte stream (~7–24 bytes typically)

### 4.5 Transfer to MAC — Bypassing PDCP/RLC

```c
f1ap_dl_rrc_message_t dl_rrc = {
    .rrc_container        = buffer,
    .rrc_container_length = length,
    .srb_id               = RRC_DL_PAGING   // 0xFF — special sentinel
};
rrc->mac_rrc.dl_rrc_message_transfer(assoc_id, &dl_rrc);
```

**Why bypass PDCP/RLC?**
- **No ciphering** — paging is public broadcast
- **No RLC segmentation** — message is small, fits in one TB
- **No UE context** — UE is IDLE, no SRB exists
- Direct RRC→MAC is faster and simpler

The special `srb_id = 0xFF` tells MAC *"this is paging, not unicast — store it, don't send to RLC."*

### 🎤 How to Speak About This

> *"At RRC, the key work was implementing the PF/PO calculation per TS 38.304 Section 7.1. The UE_ID is derived from the S-TMSI, then the Paging Frame is calculated as (T/N) times UE_ID mod N plus an offset, all modulo T. This distributes UEs evenly across paging frames. For the paging occasion within a frame, there's a second formula using i_s to spread UEs across Ns slots. I store the result in a shared array protected by a pthread mutex, because RRC and MAC run in different threads. The PCCH message itself is ASN.1 encoded and sent directly to MAC bypassing PDCP and RLC — there's no per-UE security context in IDLE mode, so ciphering and segmentation aren't needed."*

---

## 7. MAC Layer

### 5.1 Receiving Paging PDU (`mac_rrc_dl_handler.c`)

MAC's `dl_rrc_message_transfer()` handles all DL RRC messages. Paging is identified by the sentinel `srb_id == RRC_DL_PAGING (0xFF)`:

```c
if (dl_rrc->srb_id == RRC_DL_PAGING) {
    mac->pcch_pdu_size = nr_build_pcch_mac_pdu(
        mac->pcch_pdu, dl_rrc->rrc_container, dl_rrc->rrc_container_length);
    mac->pcch_pending = true;
    return;  // Do NOT send to RLC
}
```

**MAC instance stores:**
```c
uint8_t pcch_pdu[128];   // PCCH payload buffer (128 bytes max)
int     pcch_pdu_size;   // Actual payload size
bool    pcch_pending;    // Flag: scheduler should transmit
```

**Why a dedicated buffer?** Normal DL uses RLC queues. Paging arrives asynchronously and must wait for the correct PF/PO — could be up to 256 frames (~2.5 seconds) away. A dedicated buffer avoids mixing with unicast traffic.

### 5.2 Scheduler Hook (`gNB_scheduler.c`)

The main TTI scheduler runs **every slot** (every 0.5ms at 30 kHz SCS):

```c
void gNB_dlsch_ulsch_scheduler(module_id_t module_idP, frame_t frame,
                                sub_frame_t slot, ...)
{
    schedule_nr_mib(...);      // MIB (every 80ms)
    schedule_nr_sib1(...);     // SIB1 (every 160ms)
    schedule_nr_paging(...);   // Paging ← OUR CODE
    // ... PRACH, UL/DL grants, etc.
}
```

### 5.3 Paging Scheduler (`gNB_scheduler_bch.c`)

```c
void schedule_nr_paging(module_id_t module_idP, frame_t frameP,
                        sub_frame_t slotP,
                        nfapi_nr_dl_tti_request_t *DL_req,
                        nfapi_nr_tx_data_request_t *TX_req)
```

**Step 1 — Pre-flight checks:**
```c
if (!gNB_mac->pcch_pending || gNB_mac->pcch_pdu_size <= 0)
    return;                           // Nothing to send
if (dl_req->nPDUs >= NFAPI_NR_MAX_DL_TTI_PDUS)
    return;                           // PDU list full
```

**Step 2 — Match frame/slot against PF/PO:**
```c
bool css_ready   = (c->active && c->num_rbs > 0);
bool sfn_parity  = ((frameP % 2) == c->sfn_c);     // Even frames only
bool pf_match    = ((frameP % T) == PF_min);
bool po_match    = (slotP == PO);

if (!(css_ready && sfn_parity && pf_match && po_match))
    return;   // Not the right time — wait
```

This runs 2000×/sec (at 30 kHz) but returns immediately 99.99% of the time — only ~1 slot per DRX cycle actually matches.

**Step 3 — Beam allocation:**
```c
int beam_index = get_fapi_beamforming_index(gNB_mac, c->ssb_index);
NR_beam_alloc_t beam = beam_allocation_procedure(
    &gNB_mac->beam_info, frameP, slotP, beam_index, n_slots_frame);
```
Paging uses the same beam as the SSB the UE would be monitoring.

**Step 4 — Time-Domain Allocation (reuse SIB1 TDA):**
```c
int tda_index = gNB_mac->radio_config.sib1_tda;
NR_tda_info_t tda_info = get_info_from_tda_tables(
    table_type, tda_index, dmrs_TypeA_Position, true);
// Result: startSymbolIndex=2, nrOfSymbols=12 (typical)
```

**Step 5 — Build PDCCH + PDSCH PDUs:**
```c
int pdu_index = gNB_mac->pdu_index[0]++;  // Unique per-slot

nr_fill_nfapi_dl_PAGING_pdu(module_idP, NULL, gNB_mac->sched_ctrlCommon,
    dl_req, pdu_index, c, gNB_mac->pcch_pdu_size,
    tda_info.startSymbolIndex, tda_info.nrOfSymbols, beam_index);
```

**Step 6 — Attach payload to TX_req:**
```c
nfapi_nr_pdu_t *tx_req = &TX_req->pdu_list[TX_req->Number_of_PDUs];
memcpy(tx_req->TLVs[0].value.direct, gNB_mac->pcch_pdu, gNB_mac->pcch_pdu_size);
tx_req->PDU_index  = pdu_index;     // ← MUST match DL_req
tx_req->num_TLV    = 1;
tx_req->TLVs[0].length = gNB_mac->pcch_pdu_size;
TX_req->Number_of_PDUs++;
```

> [!IMPORTANT]
> **PDU_index synchronization is critical.** PHY matches DL_req (scheduling info) with TX_req (payload) using this index. Mismatch = wrong data transmitted on air.

**Step 7 — Mark consumed:**
```c
gNB_mac->pcch_pending  = false;
gNB_mac->pcch_pdu_size = 0;
```

### 5.4 DCI and PDSCH PDU Building

Inside `nr_fill_nfapi_dl_PAGING_pdu()`:

**PDCCH (DCI):**
```c
dci_pdu->RNTI             = P_RNTI;          // 0xFFFE
dci_pdu->AggregationLevel = agg_level;       // e.g., 4 CCEs
dci_pdu->CceIndex         = cce_index;
dci_pdu->ScramblingId     = physCellId;
dci_pdu->ScramblingRNTI   = 0;               // 0 for P-RNTI

// DCI payload fields:
dci_payload.frequency_domain_assignment = RIV(rbSize, rbStart, BWPSize);
dci_payload.time_domain_assignment      = tda_index;
dci_payload.mcs                         = 0;    // QPSK, lowest rate
dci_payload.system_info_indicator       = 1;    // Indicates paging/SI
```

**PDSCH:**
```c
pdsch_pdu->rnti             = P_RNTI;
pdsch_pdu->BWPSize          = 273;            // Full carrier BW
pdsch_pdu->rbStart          = 0;
pdsch_pdu->rbSize           = 2;              // Small allocation
pdsch_pdu->mcsIndex[0]      = 0;              // MCS 0 for coverage
pdsch_pdu->qamModOrder[0]   = 2;              // QPSK
pdsch_pdu->nrOfLayers       = 1;
pdsch_pdu->TBSize[0]        = pcch_pdu_size;
pdsch_pdu->dataScramblingId = physCellId;
pdsch_pdu->dlDmrsScramblingId = physCellId;
```

### 5.5 FAPI Interface (MAC→PHY)

MAC sends two messages per paging occasion:

**DL_TTI.request** — scheduling metadata:
```
┌─────────────────────────┐
│ Frame: 10, Slot: 0      │
│ nPDUs: 2                │
│ PDU[0]: PDCCH            │
│   RNTI=0xFFFE, CCE=0    │
│   DCI payload (bits)     │
│ PDU[1]: PDSCH            │
│   RNTI=0xFFFE            │
│   PDU_index=5            │
│   RB 0-1, Symbol 2-13   │
│   TBS=24 bytes           │
└─────────────────────────┘
```

**TX_Data.request** — actual payload:
```
┌─────────────────────────┐
│ Frame: 10, Slot: 0      │
│ PDU[0]:                  │
│   PDU_index=5  ← match! │
│   Data: [PCCH bytes...]  │
└─────────────────────────┘
```

PHY then: channel codes → scrambles with P-RNTI → modulates (QPSK) → maps to PRBs → beamforms → transmits.

### 🎤 How to Speak About This

> *"At MAC, the scheduler runs every slot — that's every 0.5ms at 30 kHz SCS. I added a paging check that compares the current frame and slot against the stored PF and PO. When they match, I build two FAPI messages: DL_TTI.request with PDCCH DCI using P-RNTI and PDSCH configuration, and TX_Data.request with the actual PCCH payload. The critical detail is PDU_index synchronization — PHY uses this to match scheduling info with the right payload. I reuse the same CORESET0 and TDA configuration as SIB1, which is correct per spec since both use Type0-PDCCH common search space."*

---

## 8. Wireshark Integration

### T-Tracer Modification (`macpdu2wireshark.c`)

OAI's T-tracer exports internal MAC PDUs to Wireshark for protocol analysis. Original code only knew SI-RNTI and C-RNTI:

**Before:**
```c
rnti_type = (rnti == 0xFFFF) ? NR_SI_RNTI : NR_C_RNTI;
```

**After (three-way check):**
```c
rnti_type = (rnti != 0xFFFF)
    ? ((rnti != 0xFFFE) ? NR_C_RNTI : NR_P_RNTI)
    : NR_SI_RNTI;
```

Also added `NR_P_RNTI` to the RNTI tag emission so Wireshark's MAC-NR dissector can identify and decode paging:

```c
if (rnti_type == NR_C_RNTI || rnti_type == NR_RA_RNTI || rnti_type == NR_P_RNTI) {
    PUTC(&d->buf, MAC_NR_RNTI_TAG);
    PUTC(&d->buf, (rnti >> 8) & 255);
    PUTC(&d->buf, rnti & 255);
}
```

### T-Tracer Logging in Scheduler

```c
T(T_GNB_MAC_DL_PDU_WITH_DATA,
  T_INT(module_idP), T_INT(CC_id),
  T_INT(0xFFFE),                    // P-RNTI
  T_INT(frameP), T_INT(slotP),
  T_INT(0),                         // harq_pid=0 (broadcast)
  T_BUFFER(gNB_mac->pcch_pdu, gNB_mac->pcch_pdu_size));
```

### Wireshark Result

**Without fix:** `RNTI: 0xfffe (Unknown) — Unable to decode`

**With fix:**
```
MAC-NR → RNTI: 0xfffe (P-RNTI - Paging) → Direction: Downlink
  └─ PCCH Message → Paging
       └─ PagingRecordList: 1 item
            └─ ue-Identity: ng-5G-S-TMSI: 0x12345678
```

### How to Capture

```bash
# Terminal 1: T-tracer
./macpdu2wireshark -d ../T_messages.txt -live -ip 127.0.0.1 -p 9999

# Terminal 2: Wireshark
wireshark -k -i lo -f "udp port 9999"

# Filter: mac-nr.rnti == 0xfffe
```

### 🎤 How to Speak About This


> *"I also modified OAI's Wireshark tracer to correctly handle P-RNTI. The original code had a binary check — either SI-RNTI or C-RNTI. I added a third case for P-RNTI at 0xFFFE and included it in the RNTI tag emission, so Wireshark's MAC-NR dissector can properly decode the PCCH paging message structure. This was essential for end-to-end verification — I could see the actual paging record with the UE's S-TMSI in the Wireshark decode."*

---

## 9. V2 Key Improvements (Multi-UE Patch)

V1 got the end-to-end flow working. V2 hardened it for correctness and multi-UE support. These are the **four critical improvements** to understand.

### 7.1 DCI Format 1_0 for P-RNTI — Correct Bit Packing

**The problem:** V1 reused the SI-RNTI DCI field layout, setting `system_info_indicator = 1`. But P-RNTI DCI has a **different field structure** per **TS 38.212 Table 7.3.1-2.1**.

**DCI Format 1_0 scrambled with P-RNTI (TS 38.212 §7.3.1.2.1):**

| Field | Bits | Purpose |
|-------|------|---------|
| Short Messages Indicator | 2 | `00`=both scheduling+short msg, `01`=scheduling only, `10`=short msg only |
| Short Messages | 8 | System info modification, ETWS, CMAS indicators |
| Frequency Domain Assignment | ⌈log₂(N_RB×(N_RB+1)/2)⌉ | PDSCH PRB allocation (RIV) |
| Time Domain Assignment | 4 | PDSCH symbol allocation |
| VRB-to-PRB Mapping | 1 | Interleaved (1) or non-interleaved (0) |
| MCS | 5 | Modulation and Coding Scheme |
| TB Scaling | 2 | Transport Block scaling factor |
| Reserved | 6 | Set to 0 |

**Compare with C-RNTI DCI 1_0** — which has HARQ Process Number (4b), NDI (1b), RV (2b), TPC for PUCCH (2b), PDSCH-to-HARQ timing (3b) instead. P-RNTI has **none** of these because paging is one-shot broadcast with no HARQ feedback.

**V2 code rewrites bit packing:**
```c
// V2: Correct P-RNTI bit packing (pos starts at 0)
// 1. Short Messages Indicator (2 bits)
*dci_pdu |= ((uint64_t)dci_pdu_rel15->smi & 0x3) << (dci_size - pos - 2);
pos += 2;
// 2. Short Messages (8 bits)
*dci_pdu |= ((uint64_t)dci_pdu_rel15->short_messages & 0xFF) << (dci_size - pos - 8);
pos += 8;
// 3. Frequency Domain (fsize bits)
*dci_pdu |= ((uint64_t)dci_pdu_rel15->frequency_domain_assignment.val) << (dci_size - pos - fsize);
pos += fsize;
// 4. Time Domain (4 bits)
// 5. VRB-to-PRB (1 bit)
// 6. MCS (5 bits)
// 7. TB Scaling (2 bits)
// 8. Reserved (6 bits = 0)
```

**Special case: `smi == 0b10`** (short-message-only) — all scheduling fields (freq/time/MCS) are zeroed. UE reads only the Short Messages field (ETWS/CMAS alerts).

> [!IMPORTANT]
> **Interview gold:** If asked *"What's different about DCI for P-RNTI vs C-RNTI?"*:
> - P-RNTI has **Short Message Indicator + Short Messages** (C-RNTI doesn't)
> - P-RNTI has **TB Scaling** instead of HARQ/NDI/TPC/PUCCH fields
> - **No HARQ feedback** — paging is broadcast, no ACK/NACK from UE
> - Both use same DCI size (DCI Format 1_0), but fields are completely different

### 7.2 Full 48-bit 5G-S-TMSI Construction

**The problem:** V1 passed only the raw 32-bit `m_tmsi` to PF/PO calculation. But TS 38.304 §7.1 says:

> `UE_ID = 5G-S-TMSI mod 1024`

Where **5G-S-TMSI** is 48 bits = AMF Set ID (10b) + AMF Pointer (6b) + 5G-TMSI (32b).

**V2 constructs the full identity:**
```c
uint16_t packed     = NGAP_PAGING_IND(msg_p).ue_paging_identity.s_tmsi.amf_set_id;
uint32_t fiveg_tmsi = NGAP_PAGING_IND(msg_p).ue_paging_identity.s_tmsi.m_tmsi;

// Extract fields
uint16_t amf_set_id  = (packed >> 6) & 0x3FF;   // 10 bits
uint8_t  amf_pointer = packed & 0x3F;            // 6 bits

// Build 48-bit S-TMSI as big-endian byte array
uint8_t s_tmsi_bytes[6];
s_tmsi_bytes[0] = (amf_set_id >> 2) & 0xFF;
s_tmsi_bytes[1] = ((amf_set_id & 0x3) << 6) | (amf_pointer & 0x3F);
s_tmsi_bytes[2] = (fiveg_tmsi >> 24) & 0xFF;
s_tmsi_bytes[3] = (fiveg_tmsi >> 16) & 0xFF;
s_tmsi_bytes[4] = (fiveg_tmsi >> 8)  & 0xFF;
s_tmsi_bytes[5] = fiveg_tmsi & 0xFF;

// As uint64 for PF/PO calculation
uint64_t s_tmsi_uint64 = ((uint64_t)amf_set_id << 38)
                       | ((uint64_t)amf_pointer << 32)
                       | (uint64_t)fiveg_tmsi;
```

**Impact on PF/PO:** `UE_ID = s_tmsi_uint64 mod 1024` gives a different (correct) value than `m_tmsi mod N`. This means the UE and gNB now agree on the same PF/PO.

**Impact on PCCH message:** V1 put 32-bit TMSI in PagingRecord. V2 puts the full 48-bit `ng-5G-S-TMSI` as a 6-byte BIT_STRING, which is what the spec requires and what the UE expects.

### 7.3 Multi-UE Paging Table Management

**V1 limitation:** Scheduler always reads `NR_UE_PF_PO[0][0]` — only the first entry. If UE-B's paging arrives before UE-A is scheduled, UE-A is overwritten and lost.

**V2 two-pass insert logic in RRC:**

```
Pass 1: Scan array for existing entry with same ue_index_value → UPDATE it
Pass 2: If not found, scan for first free slot → INSERT new entry
Store the chosen index in gNB_MAC_INST.freeUeIndexPaging
```

```c
// Pass 1: Update existing
for (int i = 0; i < MAX_MOBILES_PER_GNB; i++) {
    if (NR_UE_PF_PO[CC_id][i].enable_flag &&
        NR_UE_PF_PO[CC_id][i].ue_index_value == (tmsi % 1024)) {
        // UPDATE existing entry
        free_index = i;
        break;
    }
}

// Pass 2: Insert new
if (free_index < 0) {
    for (int i = 0; i < MAX_MOBILES_PER_GNB; i++) {
        if (!NR_UE_PF_PO[CC_id][i].enable_flag) {
            free_index = i;
            break;
        }
    }
}

// Tell MAC which entry to schedule
gNB_mac->freeUeIndexPaging = free_index;
```

**V2 scheduler reads the correct index:**
```c
// V1: NR_UE_PF_PO[0][0]  — hardcoded
// V2: NR_UE_PF_PO[CC_id][free_ue_index_paging]  — per-UE
```

**What would you improve further?** (Great interview follow-up)
- **Hash table** keyed by `(frame, slot)` → O(1) lookup, natural grouping
- **TTL/expiry** on entries — old paging entries cleaned up periodically
- **Queue** instead of single buffer — multiple PCCH PDUs can be pending simultaneously

### 7.4 PHY Rate Matching Fix for Small Paging TBs

**The problem:** Paging TBs are tiny (7–10 bytes, ~56–80 info bits). The LDPC rate matcher is tuned for data channels with large TBs (hundreds of bytes). For small TBs:

1. LDPC uses **Base Graph 2** (BG2) with small lifting factor Z
2. The circular buffer size `Ncb` is small
3. The filler offset `Foffset` can **exceed `Ncb`** → out-of-bounds memory access
4. After the initial rate matching loop, `k < E` (not enough coded bits to fill the output)

**V2 fix:**
```c
// Clamp Foffset to prevent OOB
if (Foffset > Ncb)
    Foffset = Ncb - 1;

// After initial copy, fill remaining with repetition
while (k < E) {
    if (d[ind] != NR_NULL) {
        e[k] = d[ind];
        k++;
    }
    ind = (ind + 1) % Ncb;  // Wrap around circular buffer
}
```

**Why this matters for interviews:** This is a **cross-layer debugging story**. A MAC-level decision (small TBS for paging) caused a PHY-layer crash (rate matcher OOB). Understanding this shows you can debug across the protocol stack.

### 🎤 How to Speak About All V2 Improvements

> *"In the second iteration, I fixed three key issues. First, the DCI bit packing — V1 reused SI-RNTI fields, but P-RNTI DCI has a completely different layout per TS 38.212. P-RNTI includes Short Message Indicator and TB Scaling fields instead of HARQ feedback fields, because paging is one-shot broadcast. Second, I fixed the UE identity — the spec says UE_ID equals the full 48-bit 5G-S-TMSI mod 1024, not just the 32-bit TMSI. I had to construct the S-TMSI from AMF Set ID, AMF Pointer, and 5G-TMSI. Third, I added multi-UE support with a two-pass insertion algorithm and per-UE index tracking. There was also an interesting PHY bug — the LDPC rate matcher crashed for small paging TBs because the filler offset exceeded the circular buffer size. I added clamping and repetition logic to handle these edge cases."*

---

## 10. Interview Q&A Bank

### Q1: Walk me through the entire paging flow end-to-end.

> *"When AMF needs to reach an IDLE UE, it sends NGAP Paging with the UE's 5G-S-TMSI and TAI list. At NGAP, I verify the TAI matches our cell's tracking area. At RRC, I compute PF and PO using the TS 38.304 formula — UE_ID is 5G-S-TMSI mod 1024, PF distributes UEs across frames in a DRX cycle, and PO distributes within frames using Ns. I encode a PCCH message with ASN.1 and send it directly to MAC with a special SRB_ID of 0xFF, bypassing PDCP and RLC. At MAC, the scheduler checks every slot whether current frame/slot matches PF/PO. When they match, it builds PDCCH DCI Format 1_0 with P-RNTI (0xFFFE) on CORESET0 and PDSCH with the paging payload, then sends both to PHY via FAPI. The UE, monitoring P-RNTI in Type0-PDCCH CSS during its PF/PO, decodes the DCI, finds the PDSCH allocation, and reads the paging record to check if its S-TMSI matches."*

---

### Q2: Why bypass PDCP/RLC for paging?

> *"Three reasons: First, no ciphering — paging is public broadcast, there's no security context. Second, no segmentation — the PCCH message is small (7–24 bytes), fits easily in one transport block. Third, no per-UE context — the UE is in IDLE mode, there's no established SRB or RLC entity. Sending directly from RRC to MAC is simpler and avoids unnecessary processing overhead."*

---

### Q3: How do you handle threading between RRC and MAC?

> *"RRC and MAC run in different threads. The shared data is the NR_UE_PF_PO array and the pcch_pdu buffer. I use a pthread mutex around the PF/PO array — RRC locks to write when paging arrives, MAC locks to read in the scheduler. The PCCH buffer uses a simpler pcch_pending boolean flag set atomically after the memcpy completes. For optimization, you could use a read-write lock since MAC mostly reads, or even a lock-free queue for zero-contention on the hot path."*

---

### Q4: Why is PF forced to even frames?

> *"Type0-PDCCH common search space, which is used for both SIB1 and paging, is only transmitted on even frames. This is a spec requirement from TS 38.213. The SSB-to-CORESET0 multiplexing pattern defines that PDCCH monitoring occasions for Type0-CSS only occur on even SFN. In V1, if the PF calculation gave an odd number, I rounded up to the next even frame."*

---

### Q5: What's different about DCI Format 1_0 for P-RNTI vs C-RNTI?

> *"Both use DCI Format 1_0 with the same total size, but the field layout is completely different. P-RNTI DCI has Short Messages Indicator (2 bits) and Short Messages (8 bits) at the beginning — for ETWS/CMAS emergency alerts. It also has TB Scaling (2 bits) instead of HARQ Process Number, NDI, RV, TPC, and PUCCH Resource Indicator. The reason: paging is one-shot broadcast with no HARQ feedback, so there's no ACK/NACK, no retransmission, no power control for PUCCH. The UE simply monitors, decodes, and either finds its TMSI or doesn't."*

---

### Q6: How does PDU_index synchronization work between DL_req and TX_req?

> *"PHY receives two FAPI messages per slot: DL_TTI.request with scheduling metadata and TX_Data.request with actual payloads. PHY matches them using PDU_index. In the scheduler, I get a unique index from gNB_mac->pdu_index, use it for both the PDSCH PDU in DL_req and the data in TX_req. If these mismatch, PHY would transmit the wrong data — for example, SIB1 payload on paging resources. In production, I'd add assertions to catch this at runtime."*

---

### Q7: What are the limitations of V1, and how does V2 fix them?

> *"V1 had three main limitations. First, single-UE — the scheduler always checked index [0][0], so only one UE's PF/PO was active. V2 adds a proper index array with two-pass insertion (update-existing then find-free). Second, wrong UE identity — V1 used raw 32-bit m_tmsi for PF/PO but the spec requires 48-bit 5G-S-TMSI mod 1024. V2 constructs the full S-TMSI from AMF Set ID, AMF Pointer, and 5G-TMSI. Third, incorrect DCI packing — V1 used SI-RNTI field layout for P-RNTI. V2 rewrites it to match TS 38.212 exactly."*

---

### Q8: What's UE_ID = 5G-S-TMSI mod 1024? Why mod 1024?

> *"Per TS 38.304 Section 7.1, 1024 is the fixed modulus that ensures UE_ID fits in 10 bits. The 5G-S-TMSI is 48 bits — you take the full identity mod 1024 to get the UE_ID. This UE_ID then feeds into the PF formula: PF = (T/N) × (UE_ID mod N) + offset. The mod 1024 ensures even distribution — any two UEs with different S-TMSIs have a high probability of mapping to different PF/PO combinations, which is essential for load balancing across paging occasions."*

---

### Q9: What happens if paging arrives but the PF is far in the future?

> *"In V1, the paging sits in the pcch_pdu buffer until the PF/PO arrives. Worst case with T=256 at 10ms frame duration is 2.56 seconds of waiting. The risk: if another paging arrives before the first is transmitted, V1 overwrites the buffer. V2 improves this with per-UE indexing. For production, I'd add a circular queue of PCCH PDUs, or a hash table keyed by (frame, slot) containing lists of UEs to page. I'd also add TTL/expiry to prevent stale entries from filling the array."*

---

### Q10: How would you test paging without real hardware?

> *"Three-layer testing. First, unit tests — verify PF/PO calculation with known TMSI values, check that the scheduler triggers on the right frame/slot and stays silent on others. Second, integration testing with UERANSIM (software UE simulator) + Open5GS core — register UE, move to IDLE, trigger paging via ping to the UE's IP, verify the full flow. Third, Wireshark verification — I modified the T-tracer to handle P-RNTI so I could verify the PCCH message content, timing, and DCI fields in Wireshark captures. For automation, I wrote a test script that checks gNB logs for 'Paging scheduled successfully' and T-tracer output for P-RNTI detection."*

---

### Q11: Why does SIB1 and paging use the same CORESET0?

> *"Per TS 38.213 Section 13, the UE monitors Type0-PDCCH CSS in CORESET0 for both SIB1 and paging. This is a design choice for UE power efficiency — the UE only needs to wake up and monitor one search space. In V2, I reuse the existing type0_PDCCH_CSS_config from SIB1 setup rather than creating a separate configuration. The scheduler avoids collisions by skipping SIB1 when paging is pending — they share the same time-frequency resources."*

---

### Q12: Explain the PHY rate matching bug for paging.

> *"Paging TBS is very small — around 7 to 10 bytes. LDPC with Base Graph 2 and a small lifting factor Z creates a small circular buffer Ncb. The rate matcher calculates a starting offset Foffset into this buffer, but for tiny TBs, Foffset can exceed Ncb entirely. This caused an out-of-bounds memory access. Additionally, after the initial puncturing and rate matching loop, there weren't enough coded bits to fill the output buffer E. The fix was two-fold: clamp Foffset to Ncb-1 if it exceeds it, and add a repetition loop that wraps around the circular buffer to fill the remaining output. This is a great example of cross-layer debugging — a MAC scheduling decision exposed a PHY edge case."*

---

### Q13: What's the channel mapping for paging?

> *"Logical channel PCCH maps to transport channel PCH, which maps to physical channels PDCCH and PDSCH. The RRC creates the PCCH message, MAC creates a MAC PDU on the PCH transport channel, and PHY transmits it. PDCCH carries DCI Format 1_0 scrambled with P-RNTI (0xFFFE) telling the UE where the PDSCH data is. PDSCH carries the actual paging payload. The UE in IDLE mode monitors P-RNTI on its calculated PF/PO and decodes accordingly."*

---

### Q14: What PF/PO configuration did you use and why?

> *"V1 used T=256 (rf256) with quarterT giving N=64, and Ns=2 giving two paging occasions per frame. V2 changed to T=128 (rf128) with oneT giving N=T=128, and Ns=1. The V2 configuration is simpler — every frame within the cycle is a potential paging frame (N=T), and there's exactly one PO per frame. This reduces paging latency because the DRX cycle is shorter (128 vs 256 frames = 1.28s vs 2.56s), at the cost of slightly higher UE power consumption. For a lab testbed, faster response was more important than battery life."*

---

### Q15: If you were to redesign this feature from scratch, what would you change?

> *"Several things. First, I'd use a hash table keyed by (frame, slot) instead of a flat array for O(1) PF/PO lookup. Second, I'd add a proper paging queue with priority support — emergency ETWS/CMAS pages should preempt regular paging. Third, I'd make the PCCH config fully configurable through the gNB config file instead of hardcoding T and Ns. Fourth, I'd add metrics — paging success rate, average scheduling latency, queue depth — for operations monitoring. Fifth, I'd implement paging DRX optimization where the gNB tracks recent paging failures and can request AMF to increase the paging area. These are all production-level improvements that the lab prototype doesn't need but a commercial product would."*

---

### Q16: How does the paging message travel from NGAP to RRC? What IPC is used?

> *"OAI uses an ITTI (Inter-Task Interface) message-passing framework. NGAP and RRC run in separate pthreads, each with its own ITTI task queue. When NGAP receives paging, it allocates a MessageDef with message type NGAP_PAGING_IND, fills in the S-TMSI and TAI list, and calls itti_send_msg_to_task to post it to RRC's queue. This is asynchronous — NGAP returns immediately. RRC picks it up from its event loop via itti_receive_msg and dispatches to the paging handler based on the message ID. The queue is backed by a linked list with a pthread condition variable for blocking."*

---

### Q17: How does RRC pass the paging PDU to MAC? Why not use ITTI there too?

> *"RRC calls MAC through a function-pointer callback — mac_rrc.dl_rrc_message_transfer(). This is a direct synchronous call because in monolithic gNB deployment, RRC and MAC are in the same process. I package the PCCH payload as a fake DL RRC message with srb_id set to 0xFF — this sentinel tells MAC it's paging, not a unicast SRB message, so MAC stores it in its pcch_pdu buffer instead of forwarding to RLC. In a CU/DU split deployment, this same callback would be replaced with an F1AP message sent over a network socket — the interface abstraction via function pointers makes this swap transparent."*

---

### Q18: What are the key data structures involved in paging, and who owns them?

> *"There are three main data structures. First, the NR_UE_PF_PO array — a 2D global array indexed by CC and UE, storing each UE's computed Paging Frame and Paging Occasion. RRC writes this, MAC reads it, protected by a pthread mutex. Second, the pcch_pdu buffer inside the MAC instance — 128 bytes storing the ASN.1-encoded PCCH payload with a pending flag. Third, the FAPI message structures — nfapi_nr_dl_tti_request for scheduling metadata and nfapi_nr_tx_data_request for the payload — these are what MAC sends to PHY every slot. The critical design choice is that PF/PO and PCCH are separate: the timing info is in the shared array (set by RRC), while the actual payload is in MAC's buffer (set via function-pointer callback), and they come together in the scheduler when the frame/slot match."*

---

### Q19: How does MAC talk to PHY for paging? Explain the FAPI interface.

> *"MAC communicates with PHY via FAPI — two messages per paging transmission. DL_TTI.request carries the scheduling metadata: a PDCCH PDU with the DCI bit payload scrambled with P-RNTI, specifying aggregation level and CCE index, plus a PDSCH PDU with RB allocation, MCS, TBS, and a PDU_index. TX_Data.request carries the actual PCCH bytes, also tagged with the same PDU_index. PHY matches them — it finds the PDSCH PDU in DL_TTI.request with PDU_index=5, then looks for the TX_Data entry with PDU_index=5 to get the payload. If these indices mismatch, PHY would transmit wrong data on air, which is why index synchronization is one of the most critical implementation details."*

---

## 11. Quick Reference Card

**Use this for last-minute review before the interview:**

| Concept | Key Fact |
|---------|----------|
| **P-RNTI** | `0xFFFE` |
| **DCI Format** | 1_0 (same format as C-RNTI/SI-RNTI but different fields) |
| **P-RNTI unique fields** | Short Messages Indicator (2b), Short Messages (8b), TB Scaling (2b) |
| **P-RNTI missing fields** | HARQ, NDI, RV, TPC, PUCCH Resource (no feedback!) |
| **UE_ID formula** | `5G-S-TMSI mod 1024` |
| **PF formula** | `((T/N) × (UE_ID mod N) + offset) % T` |
| **PO formula** | `i_s = floor((UE_ID/N) mod Ns)`, PO = `i_s × (slots_per_frame/Ns)` |
| **5G-S-TMSI** | 48 bits = AMF Set ID (10b) + AMF Pointer (6b) + 5G-TMSI (32b) |
| **Channel mapping** | PCCH → PCH → PDSCH (with PDCCH for DCI) |
| **Search space** | Type0-PDCCH CSS in CORESET0 (shared with SIB1) |
| **Why no PDCP/RLC** | Broadcast, no ciphering, no segmentation, no UE context |
| **IPC: NGAP→RRC** | ITTI async message queue (MessageDef + condvar) |
| **IPC: RRC→MAC** | Function-pointer callback (direct call, monolithic) or F1AP (CU/DU split) |
| **IPC: MAC→PHY** | FAPI: DL_TTI.request + TX_Data.request linked by PDU_index |
| **Threading** | SCTP rx → RRC task → MAC scheduler → PHY L1 (all separate pthreads) |
| **Shared DS** | `NR_UE_PF_PO[][]` (mutex-protected), `pcch_pdu[]` (flag-based) |
| **Spec refs** | PF/PO: TS 38.304 §7.1 · DCI: TS 38.212 §7.3.1.2.1 · CSS: TS 38.213 §13 |
| **V1 bug: UE_ID** | Used `m_tmsi % N` instead of `5G-S-TMSI mod 1024` |
| **V1 bug: DCI** | Used SI-RNTI field layout for P-RNTI |
| **PHY bug** | LDPC rate matcher Foffset > Ncb for tiny TBs |

---


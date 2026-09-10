# National Digital Image Sharing (NDIS) — Qatar

## 1. What is NDIS?

**NDIS** stands for **National Digital Image Sharing**.

Based on the RFP excerpt, NDIS is intended to be a **national-level capability for sharing medical images and associated diagnostic reports across Qatar's healthcare sector**.

> A patient's medical images and diagnostic reports should be securely available to authorized healthcare professionals when needed, including when the patient moves between healthcare organizations.

NDIS is therefore a **national healthcare interoperability and image-sharing initiative**—not simply a PACS used within a single hospital.

---

## 2. Client problem statement

Based on the RFP excerpts, the client is **not simply looking for a PACS, a viewer, or a cloud deployment**. The requirement is broader:

> **Qatar needs a national, secure, standards-based, highly available digital imaging ecosystem that enables healthcare providers to discover, access, exchange, view, report on and analyze medical images and diagnostic reports through a unified national platform. The platform must integrate with QHIE-Hub, operate within MOPH's Azure Qatar Central environment, support business continuity and establish a trusted foundation for national analytics and future decision support.**

### 2.1 Fragmented imaging environments

Healthcare imaging is distributed across many providers. Each may use different:

- PACS and VNA platforms
- RIS, EHR / EMR, and HIS systems
- Imaging modalities and technology vendors
- Local patient identifiers and clinical workflows
- DICOM versions, metadata conventions, and levels of FHIR maturity

The national challenge is therefore:

> **How can these heterogeneous local environments operate as one connected national ecosystem without requiring every provider to replace its existing systems?**

| At Healthcare Provider A | When the patient later visits Healthcare Provider B |
|---|---|
| MRI images, diagnostic reports, and patient/study information may remain within Provider A's PACS, RIS, and EHR environment. | The clinician may need to request records manually, contact Provider A, use a separate transfer mechanism, or potentially repeat the examination. |

### 2.2 National interoperability and image exchange

NDIS must make imaging information **discoverable and retrievable across organizations**. For a prior study, an authorized clinician must be able to determine:

1. Whether the study exists
2. Where it is located
3. Whether a diagnostic report is available
4. Whether access is authorized
5. How to retrieve and view the study safely

This makes NDIS a combined challenge of:

```text
Discovery + patient identity + interoperability + authorization + retrieval + viewing + reporting
```

The intended national integration path is:

```text
Existing healthcare-provider systems
PACS · VNA · RIS · EHR / HIS · imaging modalities
                    ↓
QHIE-Hub
                    ↓
NDIS
                    ↓
National sharing for authorized clinical users
```

The RFP requires a standards-based approach, including **FHIR R4B**, **DICOM**, **QIDO-RS**, **WADO-RS**, **C-FIND**, **C-MOVE**, and applicable national terminology standards.

### 2.3 Data and semantic standardization

Technical connectivity alone is insufficient. Data from different organizations must have consistent meaning—for example, patient identifiers, procedure names, modality names, diagnostic terms, observations, and metadata.

| Standardization layer | Client need |
|---|---|
| **Technical interoperability** | Systems exchange imaging and clinical information reliably. |
| **Semantic interoperability** | Images, reports, metadata, and clinical terms have consistent meaning across organizations. |
| **Data quality** | Patient, study, report, and source metadata are complete, accurate, and reconcilable. |

The RFP's use of **HL7 FHIR R4B**, **ICD-10-CM**, **SNOMED CT**, and **LOINC**, where applicable, addresses this semantic-interoperability requirement.

### 2.4 One national clinical experience

Clinicians should not need to know which provider produced a study, which PACS or VNA holds it, or which interface retrieves it. The client needs a consistent clinical access experience through a **zero-footprint Universal Viewer**.

Expected access contexts include:

- Browsers and desktop environments
- Smartphones and tablets
- Android, iOS, and iPadOS
- Embedded access through PHR and HIE Viewer environments

The usability challenge is:

> **How can clinicians view relevant studies and reports consistently, regardless of the source organization or source imaging system?**

### 2.5 Modern and legacy system compatibility

Providers may use both modern DICOMweb interfaces and traditional DICOM services. NDIS must bridge both generations rather than require every provider to upgrade before participating.

| Integration type | Relevant standards / services |
|---|---|
| **Modern web-based imaging exchange** | QIDO-RS and WADO-RS |
| **Traditional imaging exchange** | C-FIND and C-MOVE |
| **Clinical and event integration** | FHIR R4B and notification services |

The client also requires timely awareness of changes—such as a new study, updated report, or changed patient information—rather than relying only on slow batch synchronization.

```text
Provider updates a study or report
            ↓
FHIR notification
            ↓
QHIE-Hub / NDIS receives the update
            ↓
National information remains current
```

### 2.6 Availability, business continuity, and disaster recovery

NDIS is national healthcare infrastructure, not a conventional business application. The RFP expects high availability, redundancy, and failover across interfaces, servers, databases, applications, and supporting components.

A key requirement is continuity when critical dependencies are unavailable, including **QHIE-Hub** or the **NDIS VNA**. The client is asking not only how infrastructure recovers, but how clinical workflows can continue during partial failure.

This introduces requirements for:

- Degraded or offline operation where applicable
- Queueing, retries, local caching, and temporary storage
- Synchronization and reconciliation after recovery
- Tested continuity workflows
- Defined **RTO** and **RPO** targets
- Operational DR playbooks, scripts, processes, and test evidence

> The client wants disaster recovery to be an executable, measurable, and tested operational capability—not a theoretical document.

### 2.7 MOPH cloud governance

NDIS must be deployed within **MOPH's existing Azure Qatar Central subscription and landing zone**. The solution must align with existing cloud requirements for identity, networking, security, policy, monitoring, and operations.

The deployment challenge is:

> **How can a complex national clinical-imaging platform be delivered within a governed Azure environment while meeting MOPH operational and security standards?**

### 2.8 Data engineering, analytics, and storage lifecycle

The RFP requires application data and medical images to be transferred into the **MOPH Azure DataLakeHouse**. This is distinct from clinical, transactional image access:

| Operational NDIS environment | MOPH Azure DataLakeHouse |
|---|---|
| Patient care, image access, reporting, and transactional workflows | Analytics, reporting, historical analysis, decision support, and potential future AI use cases |

The client needs secure, reliable data extraction, transformation, validation, orchestration, monitoring, lineage, and data-quality controls—without degrading clinical performance.

Medical images also create national-scale storage pressures: high volume, rapid growth, long retention, network transfer, retrieval expectations, and cost. The RFP therefore calls for lifecycle management across hot, cool, and cold/archive storage.

> **The objective is to balance storage cost, retrieval performance, clinical accessibility, and agreed service levels—not merely to store images as cheaply as possible.**

### 2.9 National operations, data trust, and provider onboarding

The client needs end-to-end observability across the ecosystem, including provider connectivity, DICOM queries, FHIR notifications, report synchronization, QHIE-Hub and VNA health, data pipelines, archive retrieval, and Azure storage growth.

It must also prove that a transaction is complete and trustworthy:

```text
Source study
= NDIS record
= FHIR metadata
= retrievable image
= diagnostic report
```

Any mismatch must be detectable through reconciliation and data-quality processes.

As all healthcare providers in Qatar are expected to connect, onboarding must be repeatable and scalable:

```text
Assessment → Mapping → Configuration → Validation → Testing → Certification → Production monitoring
```

### 2.10 National decision support

The goal extends beyond sharing a single image between two clinicians. A trusted, standardized national imaging layer can support:

- Clinical decisions through access to relevant prior studies and reports
- Operational analysis of utilization, capacity, demand, turnaround times, and bottlenecks
- Healthcare-system planning and national performance insights
- Future predictive or AI-enabled use cases, subject to approved governance

### 2.11 The deeper business problem

> **Qatar does not want clinically valuable imaging information to remain trapped inside organizational and technology silos.**

The desired outcome is imaging information that is nationally accessible to authorized users while remaining **secure, governed, standardized, clinically reliable, highly available, and financially sustainable**.

This creates unavoidable design tensions:

| Objective | Must be balanced with |
|---|---|
| Cross-provider access | Strong privacy, security, and authorization |
| National clinical experience | Federated local systems and varied vendors |
| High performance | Cost-efficient storage and archival management |
| Standardization | Support for heterogeneous and legacy systems |
| Always-on clinical workflows | Dependencies on multiple external systems |
| Real-time care | Large-scale analytics and archival workloads |

### 2.12 π by3 internal interpretation

π by3's opportunity is **not** to replace the specialist imaging platform. It is to help the prime solution provider make it a resilient, integrated, Azure-native, observable, data-ready, and operationally sustainable national healthcare service.

> **MOPH needs to establish NDIS as a national digital imaging and diagnostic-report sharing capability across healthcare providers in Qatar. The challenge is to federate heterogeneous PACS/VNA and clinical systems through QHIE-Hub using Qatar-localized HL7 FHIR R4B, DICOMweb, and traditional DICOM standards; provide a consistent Universal Viewer and reporting experience; meet strict availability, failover, DR, and business-continuity requirements; operate within MOPH's Azure Qatar Central landing zone; and establish governed data flows into the MOPH Azure DataLakeHouse for analytics and national decision support.**

NDIS is intended to enable authorized organizations to **discover, access, and share relevant imaging information and diagnostic reports**, subject to the final architecture, policies, and access rules.

---

## 3. Simple patient journey

### Step 1 — Imaging is performed

Patient X visits **Healthcare Provider A**. A physician orders a CT scan.

- The CT scanner generates medical images.
- A radiologist reviews the images.
- The radiologist produces a diagnostic report.

**Provider A holds: Patient X → CT images + diagnostic report.**

### Step 2 — Information exists within Provider A

The imaging study may flow through Provider A's own clinical environment:

```text
CT Scanner → PACS → RIS → EHR / HIS
```

### Step 3 — The patient visits another provider

Patient X later visits **Healthcare Provider B**. The clinician needs to review the previous CT scan.

### Step 4 — NDIS supports national sharing

```text
Healthcare Provider A
CT images + diagnostic report
             ↓
NDIS — National Digital Image Sharing
             ↓
Healthcare Provider B
Authorized clinician accesses the relevant previous study
```

The clinician can use existing diagnostic information when making decisions about the patient's care.

---

## 4. What NDIS shares

### Medical images

Potential imaging studies include:

- X-ray
- CT
- MRI
- Ultrasound
- Mammography
- PET/CT
- Other diagnostic imaging within the RFP scope

Medical imaging commonly uses **DICOM (Digital Imaging and Communications in Medicine)** for images and associated imaging information.

### Diagnostic reports

Images alone may not provide complete clinical context. A radiologist or other specialist interprets the study and produces a report.

```text
CT images + radiologist's diagnostic interpretation = more complete clinical context
```

NDIS focuses on **medical images and diagnostic reports**, not images alone.

---

## 5. Why “national” matters

A conventional PACS generally supports imaging operations within one hospital or healthcare organization. NDIS has the broader purpose of enabling sharing across the **national healthcare ecosystem**.

```text
Healthcare Provider A  ↘
Healthcare Provider B   →
Healthcare Provider C   →  NDIS
Diagnostic Centre       →
Other authorized entities ↗
```

The exact participating organizations must be confirmed from the full RFP and implementation documentation.

---

## 6. NDIS does not necessarily mean one national PACS

“National Digital Image Sharing” does not automatically mean that every image from every provider is moved into one central database. The eventual Qatar architecture could follow different models.

| Architecture model | Conceptual model |
|---|---|
| **Centralized** | Providers send images to a central national repository, which serves authorized consumers. |
| **Federated** | Images remain with participating providers; national services help authorized users discover and retrieve them. |
| **Hybrid** | Some information or services are centralized while some imaging content remains distributed. |

> The actual NDIS architecture—centralized, federated, or hybrid—must be confirmed from the complete RFP. It should not be inferred from a short excerpt.

---

## 7. NDIS and Qatar's QHIE-Hub

NDIS will integrate with **Qatar's QHIE-Hub** to provide a national service for exchanging medical images and diagnostic reports across the healthcare ecosystem.

```text
Healthcare providers and imaging systems
PACS · RIS · EHR / EMR · HIS · imaging modalities
                    ↓
QHIE-Hub + NDIS
National interoperability and image-sharing service
                    ↓
Authorized healthcare professionals and organizations
```

### Expected role of the integrated service

1. **National exchange**
   - Enable authorized healthcare organizations to exchange medical images and diagnostic reports.
   - Support access to relevant imaging information across the healthcare ecosystem.

2. **Standardization and interoperability**
   - Standardize images, diagnostic reports, and metadata across participating organizations.
   - Support interoperable workflows between different imaging and clinical systems.

3. **Compliance and governance**
   - Apply national requirements for security, privacy, access control, auditability, and data governance.
   - Help ensure that shared information is managed consistently across participating organizations.

4. **Trusted imaging information layer**
   - Establish a trusted, standardized imaging-information layer beyond simple data exchange.
   - Create a reliable foundation for appropriately authorized use of imaging information.

5. **Decision support**
   - Support clinical decision-making through access to relevant prior images and reports.
   - Support operational decision-making through standardized, governed imaging information.
   - Support healthcare-system-level decision-making where approved data and governance arrangements permit.

> NDIS is intended to go beyond exchanging files: integrated with QHIE-Hub, it can create a trusted and standardized imaging-information layer for clinical, operational, and healthcare-system-level decision support.

---

## 8. NDIS is an interoperability challenge

Participating organizations may use different systems, vendors, identifiers, and workflows:

- PACS
- RIS
- EHR / EMR
- HIS
- VNA
- Imaging modalities
- Patient identifiers
- Clinical and operational workflows

NDIS must allow these environments to work together according to national requirements.

| Interoperability concept | Purpose |
|---|---|
| **DICOM** | Standard for medical imaging communication and related imaging information. |
| **HL7** | Standard for exchanging healthcare information between clinical systems. |
| **FHIR** | Modern, API-oriented healthcare interoperability standard. |
| **IHE profiles** | Standardized integration workflows for healthcare systems. |

The RFP should be treated as the source of truth for mandatory standards, profiles, and interfaces.

---

## 9. Patient identity and data quality

Reliable patient matching is critical. One provider may know a patient as `Patient ID: 12345`, while another uses a different local identifier for the same individual.

Incorrect matching could have serious clinical consequences. Depending on the NDIS design, key capabilities may include:

1. Patient identity and demographic matching
2. Study and accession identification
3. Metadata quality and validation
4. Source-organization identification
5. Data-validation and exception-management processes

> The correct image must be retrieved for the correct patient, with sufficient context for safe clinical use.

---

## 10. Security, privacy, and governance

Medical images and diagnostic reports are sensitive healthcare information. NDIS requires strong controls; it cannot make all information universally accessible.

| Control area | Key question |
|---|---|
| **Authentication** | Who is accessing the system? |
| **Authorization** | Is this user allowed to access this patient's information? |
| **Access control** | Which records can this user or organization view? |
| **Encryption** | Is information protected in transit and at rest? |
| **Audit trails** | Who accessed what information, and when? |
| **Privacy and consent** | Is patient information handled according to applicable requirements? |

The precise security, privacy, consent, and access-control model must be taken from the RFP and applicable Qatar requirements.

---

## 11. Potential healthcare value

### Continuity of care

```text
Patient has an MRI at Provider A
        ↓
Patient later receives treatment at Provider B
        ↓
Provider B accesses relevant prior imaging through NDIS
        ↓
Clinician has additional clinical context for care decisions
```

NDIS can help diagnostic information follow the patient's care journey rather than remaining isolated within a single institution.

### Potential reduction in duplicate imaging

If a clinician can access a recent, relevant prior study and report, they can first assess whether the existing information is sufficient before considering another examination.

Potential benefits may include:

- Less unnecessary duplication
- Lower resource utilization
- Reduced patient inconvenience
- Faster clinical decisions

These benefits should be stated as RFP-supported outcomes only when confirmed in the complete document.

### Better-informed decisions over time

Access to prior imaging can support longitudinal comparison:

```text
Previous CT  ↔  Current CT
```

This gives clinicians context not only about the patient's condition today, but also about how it may have changed over time.

---

## 12. More than file transfer

NDIS is not simply a process where Hospital A sends an image file to Hospital B. It is a national digital-health capability involving:

- Patient identification
- Image discovery
- Image retrieval and exchange
- Diagnostic-report availability
- Metadata management
- Interoperability
- Clinical workflow integration
- Security and authorization
- Auditability
- Governance
- Availability and performance

---

## 13. Relationship to existing hospital systems

NDIS does not necessarily replace the systems providers already use. A provider may continue operating its own imaging and clinical systems while connecting them to the national sharing ecosystem.

```text
CT / MRI / X-ray
        ↓
Hospital PACS / imaging environment
        ↓
NDIS interoperability and sharing layer
        ↓
Other authorized healthcare organization
        ↓
Clinician
```

---

## 14. NDIS vs. PACS vs. syngo Carbon

| Term | Meaning |
|---|---|
| **PACS** | Technology used to store, retrieve, manage, and view medical images, traditionally within an imaging or hospital environment. |
| **syngo Carbon** | Siemens Healthineers' enterprise imaging and clinical data-management ecosystem. See [[NPC#syngo Carbon — Qatar Healthcare Context]]. |
| **NDIS** | Qatar's national initiative/capability for sharing medical images and diagnostic reports across the health sector. |

**NDIS ≠ PACS**

**NDIS ≠ syngo Carbon**

A platform such as syngo Carbon could potentially form part of, or integrate with, an NDIS architecture. Its actual role should only be stated after confirmation from the RFP or implementation documentation.

---

## 15. Strategic context for Qatar

The RFP positions NDIS as supporting priorities across Qatar's national healthcare transformation agenda:

```text
Qatar National Vision 2030
        ↓
National Development Strategy
        ↓
National Health Strategy
        ↓
National eHealth and Data Management Strategy
        ↓
NDIS
        ↓
National sharing of medical images and diagnostic reports
        ↓
Improved health outcomes
```

This positions NDIS as a strategic digital-health transformation initiative, not merely an imaging-system procurement.

---

## 16. Short definition

> **National Digital Image Sharing (NDIS)** is a Qatar national digital-health initiative intended to enable the secure sharing of medical images and associated diagnostic reports across the healthcare sector. Its purpose is to improve the availability and continuity of diagnostic imaging information across participating healthcare organizations and support better healthcare delivery and health outcomes. NDIS is positioned as a strategic enabler aligned with Qatar National Vision 2030, the National Development Strategy, National Health Strategy, and National eHealth and Data Management Strategy. Technically, it is a national image-sharing and interoperability ecosystem connecting healthcare organizations and their existing imaging and clinical environments—not simply another hospital PACS.

## 17. One-line example

> **Patient receives an MRI at Healthcare Provider A → the images and report are made discoverable/shareable through NDIS → an authorized clinician at Healthcare Provider B accesses the relevant previous study → the clinician uses it to support the patient's ongoing care.**

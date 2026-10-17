# Smart Clinic Management System: Database Architecture & Schema Design

## 1. Introduction & Architecture Rationale

When designing the **Smart Clinic Management System**, choosing the right database for each entity is critical:

* **MySQL (Relational)** is chosen for core transactional, highly structured data requiring strict ACID compliance, relational integrity, and foreign key constraints (e.g., patient records, doctor profiles, scheduling, and billing).

* **MongoDB (Document-based)** is chosen for semi-structured, highly dynamic, or high-velocity data where schemas evolve, nested arrays are beneficial, and rapid feature iteration is required (e.g., prescriptions with varying medications, specialty-specific vital signs, chat logs, and medical document metadata).

## MySQL Database Design

The relational database handles core master data, appointment scheduling constraints, and financial transactions.

```sql
-- ==========================================
-- 1. Patients Table
-- ==========================================
CREATE TABLE patients (
    patient_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender ENUM('Male', 'Female', 'Other', 'Prefer not to say') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ==========================================
-- 2. Doctors Table
-- ==========================================
CREATE TABLE doctors (
    doctor_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    specialization VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ==========================================
-- 3. Doctor Availability Table
-- ==========================================
CREATE TABLE doctor_availability (
    availability_id INT AUTO_INCREMENT PRIMARY KEY,
    doctor_id INT NOT NULL,
    day_of_week ENUM('Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday') NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    FOREIGN KEY (doctor_id) REFERENCES doctors(doctor_id) ON DELETE CASCADE
);

-- ==========================================
-- 4. Appointments Table
-- ==========================================
CREATE TABLE appointments (
    appointment_id INT AUTO_INCREMENT PRIMARY KEY,
    patient_id INT NOT NULL,
    doctor_id INT NOT NULL,
    appointment_time DATETIME NOT NULL,
    status ENUM('Scheduled', 'Confirmed', 'Completed', 'Cancelled') DEFAULT 'Scheduled',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id) ON DELETE CASCADE,
    FOREIGN KEY (doctor_id) REFERENCES doctors(doctor_id) ON DELETE CASCADE
);

-- ==========================================
-- 5. Payments Table
-- ==========================================
CREATE TABLE payments (
    payment_id INT AUTO_INCREMENT PRIMARY KEY,
    appointment_id INT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    payment_status ENUM('Pending', 'Completed', 'Refunded', 'Failed') DEFAULT 'Pending',
    transaction_reference VARCHAR(100) UNIQUE,
    paid_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (appointment_id) REFERENCES appointments(appointment_id) ON DELETE CASCADE
);
```

### Design Notes for MySQL:

* **Foreign Key Constraints (`ON DELETE CASCADE`)**: Ensures data consistency so orphaned records aren't left behind if a patient or doctor profile is removed.
* **Enums**: Used for constrained fields like `gender`, `status`, and `day_of_week` to enforce valid input ranges at the database level.

## MongoDB Collection Design

MongoDB stores clinical artifacts, communications, and flexible medical records. Each document links back to MySQL relational entities using foreign key IDs (`patient_id`, `doctor_id`, `appointment_id`).

### A. `prescriptions` Collection

Prescriptions contain an array of variable medications, dosages, and instructions, making a document structure ideal for retrieving a complete prescription in a single query.

```json
{
  "_id": ObjectId("64a2f1e2c9e77b2184123456"),
  "prescription_id": "PRX-2026-8942",
  "appointment_id": 1042,
  "patient_id": 15,
  "doctor_id": 3,
  "issued_date": ISODate("2026-10-17T10:30:00Z"),
  "diagnosis": "Acute Bronchitis",
  "medications": [
    {
      "name": "Amoxicillin",
      "dosage": "500mg",
      "frequency": "3 times a day",
      "duration": "7 days",
      "instructions": "Take with food."
    },
    {
      "name": "Ibuprofen",
      "dosage": "400mg",
      "frequency": "As needed for pain/fever",
      "duration": "5 days",
      "instructions": "Do not exceed 3 tablets in 24 hours."
    }
  ],
  "doctor_notes": "Patient advised to rest and stay hydrated. Follow up if symptoms persist."
}
```

### B. `consultation_notes` Collection

Different medical specialties record different vital signs and physical observations. MongoDB's flexible schema handles varying nested vital sign structures without altering table schemas.

```json
{
  "_id": ObjectId("64a2f3b8c9e77b2184123789"),
  "appointment_id": 1042,
  "patient_id": 15,
  "doctor_id": 3,
  "created_at": ISODate("2026-10-17T10:15:00Z"),
  "chief_complaint": "Persistent cough and mild fever for 3 days.",
  "vitals": {
    "blood_pressure": "120/80",
    "heart_rate_bpm": 78,
    "temperature_celsius": 38.2,
    "weight_kg": 70.5,
    "oxygen_saturation_percent": 98
  },
  "clinical_notes": "Lungs clear on auscultation, slight wheezing in upper lobes.",
  "tags": ["respiratory", "fever", "follow-up-required"]
}
```

### C. `chat_records` Collection

Telehealth chat transcripts are stored as append-only document arrays per session, ensuring fast read and write performance during live consultations.

```json
{
  "_id": ObjectId("64a2f54ac9e77b2184123abc"),
  "appointment_id": 1042,
  "participants": {
    "patient_id": 15,
    "doctor_id": 3
  },
  "session_status": "Closed",
  "messages": [
    {
      "sender_type": "Patient",
      "sender_id": 15,
      "timestamp": ISODate("2026-10-17T09:00:00Z"),
      "message": "Hello doctor, can I take my cough syrup with food?"
    },
    {
      "sender_type": "Doctor",
      "sender_id": 3,
      "timestamp": ISODate("2026-10-17T09:02:00Z"),
      "message": "Yes, it is recommended to take it right after meals to avoid stomach upset."
    }
  ]
}
```

### D. `medical_documents` Collection

Tracks metadata for uploaded diagnostic files (X-rays, PDFs, lab reports) stored in cloud object storage, along with extracted key-value metadata.

```json
{
  "_id": ObjectId("64a2f771c9e77b2184123def"),
  "patient_id": 15,
  "uploaded_by_role": "Patient",
  "document_title": "Complete Blood Count (CBC)",
  "file_type": "application/pdf",
  "storage_url": "s3://smart-clinic-bucket/patients/15/cbc_oct2026.pdf",
  "uploaded_at": ISODate("2026-10-16T14:20:00Z"),
  "extracted_metadata": {
    "lab_name": "Metro Diagnostics",
    "white_cell_count": "7.5 x10^3/uL",
    "hemoglobin": "14.2 g/dL"
  }
}
```

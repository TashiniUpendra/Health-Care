# Health-Care
       -- Basic users for authentication (not PHI)
CREATE TABLE "users" (
  user_id SERIAL PRIMARY KEY,
  username VARCHAR(150) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(50) NOT NULL, -- e.g. admin, doctor, nurse, receptionist, patient
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  last_login TIMESTAMP WITH TIME ZONE
);

-- Patients (PHI)
CREATE TABLE "patients" (
  patient_id SERIAL PRIMARY KEY,
  external_patient_id VARCHAR(50) UNIQUE, -- optional MRN
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  dob DATE,
  gender VARCHAR(20),
  phone VARCHAR(30),
  email VARCHAR(255),
  address TEXT,
  emergency_contact_name VARCHAR(200),
  emergency_contact_phone VARCHAR(50),
  insurance_provider VARCHAR(200),
  insurance_policy_no VARCHAR(100),
  created_by INTEGER REFERENCES users(user_id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Staff / providers
CREATE TABLE "providers" (
  provider_id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(user_id) UNIQUE, -- link to users table
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  title VARCHAR(100), -- Dr, NP, etc
  specialty VARCHAR(150),
  phone VARCHAR(30),
  email VARCHAR(255),
  license_no VARCHAR(100),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Appointments
CREATE TABLE "appointments" (
  appointment_id SERIAL PRIMARY KEY,
  patient_id INTEGER REFERENCES patients(patient_id) NOT NULL,
  provider_id INTEGER REFERENCES providers(provider_id) NOT NULL,
  location VARCHAR(200),
  start_time TIMESTAMP WITH TIME ZONE NOT NULL,
  end_time TIMESTAMP WITH TIME ZONE,
  status VARCHAR(50) NOT NULL DEFAULT 'scheduled', -- scheduled, checked_in, canceled, completed, no_show
  reason TEXT,
  created_by INTEGER REFERENCES users(user_id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
CREATE INDEX idx_appointments_patient_time ON appointments(patient_id, start_time);
CREATE INDEX idx_appointments_provider_time ON appointments(provider_id, start_time);

-- Clinical encounters / notes (one encounter per visit)
CREATE TABLE "encounters" (
  encounter_id SERIAL PRIMARY KEY,
  appointment_id INTEGER REFERENCES appointments(appointment_id),
  patient_id INTEGER REFERENCES patients(patient_id) NOT NULL,
  provider_id INTEGER REFERENCES providers(provider_id) NOT NULL,
  encounter_type VARCHAR(100), -- e.g., consultation, follow-up, emergency
  summary TEXT,
  vitals JSONB, -- structured vitals: {"height_cm":180, "weight_kg":75, "bp":"120/80"}
  diagnosis TEXT,
  created_by INTEGER REFERENCES users(user_id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
CREATE INDEX idx_encounters_patient ON encounters(patient_id);

-- Medical records (documents, labs, images metadata)
CREATE TABLE "medical_records" (
  record_id SERIAL PRIMARY KEY,
  patient_id INTEGER REFERENCES patients(patient_id) NOT NULL,
  encounter_id INTEGER REFERENCES encounters(encounter_id),
  record_type VARCHAR(100), -- note, lab, image, referral
  title VARCHAR(255),
  content TEXT, -- textual content (or summary)
  metadata JSONB, -- e.g. {"lab":"CBC", "result_file":"s3://..."}
  created_by INTEGER REFERENCES users(user_id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Medications / formulary
CREATE TABLE "medications" (
  med_id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  generic_name VARCHAR(255),
  manufacturer VARCHAR(255),
  form VARCHAR(100), -- tablet, syrup, injection
  strength VARCHAR(100)
);

-- Prescriptions
CREATE TABLE "prescriptions" (
  prescription_id SERIAL PRIMARY KEY,
  patient_id INTEGER REFERENCES patients(patient_id) NOT NULL,
  provider_id INTEGER REFERENCES providers(provider_id) NOT NULL,
  med_id INTEGER REFERENCES medications(med_id) NOT NULL,
  dose VARCHAR(100),
  frequency VARCHAR(100),
  duration VARCHAR(100),
  notes TEXT,
  issued_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  status VARCHAR(50) DEFAULT 'active' -- active, completed, canceled
);
CREATE INDEX idx_prescriptions_patient ON prescriptions(patient_id);

-- Billing: invoices and payments
CREATE TABLE "invoices" (
  invoice_id SERIAL PRIMARY KEY,
  patient_id INTEGER REFERENCES patients(patient_id) NOT NULL,
  appointment_id INTEGER REFERENCES appointments(appointment_id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  total_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
  status VARCHAR(50) DEFAULT 'unpaid' -- unpaid, paid, partially_paid, canceled
);

CREATE TABLE "invoice_items" (
  item_id SERIAL PRIMARY KEY,
  invoice_id INTEGER REFERENCES invoices(invoice_id) ON DELETE CASCADE,
  description TEXT,
  amount NUMERIC(12,2) NOT NULL,
  quantity INTEGER DEFAULT 1
);

CREATE TABLE "payments" (
  payment_id SERIAL PRIMARY KEY,
  invoice_id INTEGER REFERENCES invoices(invoice_id),
  amount NUMERIC(12,2) NOT NULL,
  method VARCHAR(50), -- cash, card, insurance
  transaction_ref VARCHAR(255),
  paid_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  received_by INTEGER REFERENCES users(user_id)
);

-- Audit log
CREATE TABLE "audit_logs" (
  log_id BIGSERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(user_id),
  action VARCHAR(255),
  object_type VARCHAR(100),
  object_id VARCHAR(100),
  details JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
CREATE INDEX idx_audit_user_time ON audit_logs(user_id, created_at);


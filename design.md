# Design Document: SwastSetu AI Health Copilot

## Executive Summary

This design document outlines the technical architecture and implementation strategy for SwastSetu AI, a dual-mode AI-powered healthcare system serving rural and underserved populations across India. The system comprises Mode A (patient-facing voice-first health risk navigator) and Mode B (doctor-facing clinical intelligence cockpit), built on a microservices architecture with a 5-layer safety framework ensuring no AI-generated diagnoses or prescriptions reach patients without human oversight.

The design prioritizes safety, scalability, and accessibility, supporting 10,000 concurrent IVR calls, 22+ Indian languages, and deployment across 100+ Primary Health Centers. The architecture leverages AWS cloud infrastructure, AWS Bedrock for AI capabilities, and integrates with India's Ayushman Bharat Digital Mission (ABDM) for national health record interoperability.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SwastSetu AI System                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────────────┐         ┌──────────────────────┐         │
│  │      Mode A          │         │      Mode B          │         │
│  │  Patient Interface   │         │  Provider Interface  │         │
│  │  (IVR + Voice)       │         │  (Web + Mobile)      │         │
│  └──────────┬───────────┘         └──────────┬───────────┘         │
│             │                                 │                      │
│             └─────────────┬───────────────────┘                      │
│                           │                                          │
│              ┌────────────▼────────────┐                            │
│              │   API Gateway Layer     │                            │
│              │  (Authentication/Rate   │                            │
│              │   Limiting/Routing)     │                            │
│              └────────────┬────────────┘                            │
│                           │                                          │
│       ┌───────────────┼───────────────┬──────────────────┐         │
│       │               │               │                  │         │
│  ┌────▼─────┐  ┌─────▼──────┐  ┌────▼──────┐  ┌───────▼──────┐  │
│  │ Patient  │  │  Clinical  │  │   AI      │  │  Integration │  │
│  │ Service  │  │  Service   │  │  Service  │  │   Service    │  │
│  └────┬─────┘  └─────┬──────┘  └────┬──────┘  └───────┬──────┘  │
│       │              │               │                  │         │
│       └──────────────┼───────────────┼──────────────────┘         │
│                      │               │                            │
│              ┌───────▼───────────────▼────────┐                  │
│              │   Data Access Layer            │                  │
│              │  (Repository Pattern)          │                  │
│              └───────┬────────────────────────┘                  │
│                      │                                            │
│       ┌──────────────┼──────────────┬─────────────┐             │
│       │              │              │             │             │
│  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐  ┌───▼──────┐      │
│  │ Patient  │  │ Clinical │  │  Audit   │  │  Cache   │      │
│  │   DB     │  │   DB     │  │   DB     │  │  (Redis) │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │         External Integrations                           │  │
│  │  - AWS Bedrock (AI Models)                             │  │
│  │  - ABDM APIs (Health Records)                          │  │
│  │  - Telecom Gateway (IVR)                               │  │
│  │  - SMS/Notification Services                           │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

**Frontend:**
- Mode A: IVR system with DTMF fallback
- Mode B Web: React 18+ with TypeScript, Material-UI
- Mode B Mobile: React Native for iOS/Android

**Backend:**
- Runtime: Node.js 20+ with TypeScript
- Framework: NestJS for microservices architecture
- API: GraphQL for Mode B, REST for Mode A and integrations
- Authentication: OAuth 2.0 with JWT tokens

**AI/ML:**
- Platform: AWS Bedrock
- Models: Claude 3 for NLP, Whisper for speech-to-text
- Translation: IndicTrans2 for Indian languages
- Risk Classification: Custom fine-tuned model on medical data

**Data Storage:**
- Primary Database: PostgreSQL 15+ with TimescaleDB extension
- Document Store: MongoDB for unstructured medical documents
- Cache: Redis 7+ for session management and performance
- Object Storage: AWS S3 for audio recordings, images, documents

**Infrastructure:**
- Cloud: AWS (multi-region deployment)
- Containers: Docker with Kubernetes (EKS)
- Message Queue: AWS SQS for async processing
- CDN: CloudFront for static assets
- Monitoring: CloudWatch, Prometheus, Grafana

**Security:**
- Encryption: AES-256-GCM at rest, TLS 1.3 in transit
- Key Management: AWS KMS with HSM
- Secrets: AWS Secrets Manager
- WAF: AWS WAF for DDoS protection


## Component Design

### 1. Patient Service (Mode A)

**Responsibilities:**
- Handle IVR call flow and voice interactions
- Manage patient symptom collection and assessment
- Execute risk classification with safety guardrails
- Coordinate with translation and voice processing services

**Key Components:**

**1.1 IVR Controller**
```typescript
interface IVRController {
  handleIncomingCall(callId: string, phoneNumber: string): Promise<CallSession>;
  processVoiceInput(sessionId: string, audioData: Buffer): Promise<VoiceResponse>;
  processDTMFInput(sessionId: string, digits: string): Promise<DTMFResponse>;
  handleLanguageSelection(sessionId: string, languageCode: string): Promise<void>;
  endCall(sessionId: string, reason: CallEndReason): Promise<CallSummary>;
}

interface CallSession {
  sessionId: string;
  patientId?: string;
  phoneNumber: string;
  language: LanguageCode;
  state: CallState;
  startTime: Date;
  symptoms: Symptom[];
  conversationHistory: Message[];
}

enum CallState {
  LANGUAGE_SELECTION = 'language_selection',
  CONSENT_COLLECTION = 'consent_collection',
  SYMPTOM_COLLECTION = 'symptom_collection',
  FOLLOW_UP_QUESTIONS = 'follow_up_questions',
  RISK_ASSESSMENT = 'risk_assessment',
  RECOMMENDATION = 'recommendation',
  COMPLETED = 'completed',
  ERROR = 'error'
}
```

**1.2 Symptom Collector**
```typescript
interface SymptomCollector {
  extractSymptoms(text: string, language: LanguageCode): Promise<Symptom[]>;
  generateFollowUpQuestions(symptoms: Symptom[], history: MedicalHistory): Promise<Question[]>;
  validateSymptomCompleteness(symptoms: Symptom[]): ValidationResult;
  detectContradictions(symptoms: Symptom[]): Contradiction[];
}

interface Symptom {
  id: string;
  name: string;
  severity: SeverityLevel; // 1-10 scale
  duration: Duration;
  onset: OnsetType; // sudden, gradual
  location?: BodyLocation;
  associatedSymptoms: string[];
  aggravatingFactors?: string[];
  relievingFactors?: string[];
  confidence: number; // 0-1
}

interface Question {
  id: string;
  text: string;
  type: QuestionType; // yes_no, multiple_choice, numeric, descriptive
  options?: string[];
  priority: number; // 1-5, higher = more important
  medicalRationale: string;
}
```

**1.3 Risk Classifier**
```typescript
interface RiskClassifier {
  classifyRisk(assessment: PatientAssessment): Promise<RiskClassification>;
  explainClassification(classification: RiskClassification): Explanation;
  calculateConfidence(assessment: PatientAssessment): number;
}

interface RiskClassification {
  level: RiskLevel; // LOW, MODERATE, HIGH
  confidence: number; // 0-1
  reasoning: string[];
  contributingFactors: Factor[];
  recommendedAction: RecommendedAction;
  urgencyScore: number; // 0-100
  requiresHumanReview: boolean;
}

enum RiskLevel {
  LOW = 'low',
  MODERATE = 'moderate',
  HIGH = 'high'
}

interface RecommendedAction {
  type: ActionType; // home_care, phc_visit, emergency
  timeframe: string; // "immediately", "within 2 hours", "within 24 hours"
  nearestFacility?: HealthFacility;
  instructions: string[];
}
```

**1.4 Safety Guardrail**
```typescript
interface SafetyGuardrail {
  validateOutput(content: string, context: OutputContext): Promise<ValidationResult>;
  blockProhibitedContent(content: string): string;
  detectDiagnosis(text: string): DiagnosisDetection;
  detectMedication(text: string): MedicationDetection;
  logIntervention(intervention: SafetyIntervention): Promise<void>;
}

interface ValidationResult {
  isValid: boolean;
  blockedContent: BlockedContent[];
  sanitizedOutput: string;
  interventionRequired: boolean;
}

interface BlockedContent {
  type: ProhibitedContentType; // diagnosis, medication, dosage
  originalText: string;
  location: TextLocation;
  confidence: number;
}

enum ProhibitedContentType {
  DIAGNOSIS = 'diagnosis',
  MEDICATION = 'medication',
  DOSAGE = 'dosage',
  MEDICAL_PROCEDURE = 'medical_procedure'
}

interface SafetyIntervention {
  timestamp: Date;
  sessionId: string;
  contentType: ProhibitedContentType;
  originalContent: string;
  replacementContent: string;
  confidence: number;
}
```


### 2. Voice Processing Service

**Responsibilities:**
- Convert speech to text using Whisper/ASR
- Handle audio quality issues and noise reduction
- Support 22+ Indian languages with accent variations
- Provide confidence scoring for transcriptions

**Key Components:**

**2.1 Speech-to-Text Engine**
```typescript
interface SpeechToTextEngine {
  transcribe(audio: AudioBuffer, language: LanguageCode): Promise<Transcription>;
  detectLanguage(audio: AudioBuffer): Promise<LanguageDetection>;
  enhanceAudio(audio: AudioBuffer): Promise<AudioBuffer>;
  validateAudioQuality(audio: AudioBuffer): AudioQualityMetrics;
}

interface Transcription {
  text: string;
  confidence: number;
  language: LanguageCode;
  duration: number; // milliseconds
  alternatives?: TranscriptionAlternative[];
  wordTimestamps?: WordTimestamp[];
}

interface AudioQualityMetrics {
  signalToNoiseRatio: number;
  clarity: number; // 0-1
  isAcceptable: boolean;
  issues: AudioIssue[];
}

enum AudioIssue {
  LOW_VOLUME = 'low_volume',
  HIGH_NOISE = 'high_noise',
  DISTORTION = 'distortion',
  CLIPPING = 'clipping',
  POOR_CONNECTION = 'poor_connection'
}
```

**2.2 Text-to-Speech Engine**
```typescript
interface TextToSpeechEngine {
  synthesize(text: string, language: LanguageCode, voice: VoiceProfile): Promise<AudioBuffer>;
  getAvailableVoices(language: LanguageCode): VoiceProfile[];
  adjustSpeechRate(audio: AudioBuffer, rate: number): Promise<AudioBuffer>;
}

interface VoiceProfile {
  id: string;
  language: LanguageCode;
  gender: 'male' | 'female' | 'neutral';
  age: 'young' | 'middle' | 'senior';
  style: 'professional' | 'friendly' | 'calm';
}
```

### 3. Translation Service

**Responsibilities:**
- Translate between English and 22+ Indian languages
- Maintain medical terminology accuracy
- Handle cultural context and regional variations
- Provide confidence scoring for translations

**Key Components:**

**3.1 Translation Engine**
```typescript
interface TranslationEngine {
  translate(text: string, from: LanguageCode, to: LanguageCode): Promise<Translation>;
  translateBatch(texts: string[], from: LanguageCode, to: LanguageCode): Promise<Translation[]>;
  validateMedicalAccuracy(translation: Translation): Promise<AccuracyScore>;
  getCulturalContext(term: string, language: LanguageCode): Promise<CulturalContext>;
}

interface Translation {
  originalText: string;
  translatedText: string;
  sourceLanguage: LanguageCode;
  targetLanguage: LanguageCode;
  confidence: number;
  alternatives?: string[];
  medicalTerms: MedicalTerm[];
  requiresReview: boolean;
}

interface MedicalTerm {
  term: string;
  translation: string;
  standardTerm: string; // medical standard terminology
  culturalVariant?: string; // regional/cultural alternative
  confidence: number;
}

enum LanguageCode {
  EN = 'en', // English
  HI = 'hi', // Hindi
  BN = 'bn', // Bengali
  TE = 'te', // Telugu
  MR = 'mr', // Marathi
  TA = 'ta', // Tamil
  GU = 'gu', // Gujarati
  KN = 'kn', // Kannada
  ML = 'ml', // Malayalam
  OR = 'or', // Odia
  PA = 'pa', // Punjabi
  // ... additional languages
}
```


### 4. Clinical Service (Mode B)

**Responsibilities:**
- Manage doctor-facing triage queue and patient records
- Generate AI-powered SOAP notes and clinical suggestions
- Handle patient history and longitudinal data
- Process lab reports and medical documents

**Key Components:**

**4.1 Triage Queue Manager**
```typescript
interface TriageQueueManager {
  getQueue(facilityId: string, filters?: QueueFilters): Promise<TriageQueue>;
  addPatient(patient: PatientCase): Promise<void>;
  updatePatientStatus(caseId: string, status: CaseStatus): Promise<void>;
  notifyHighRiskCase(caseId: string): Promise<void>;
  reorderQueue(facilityId: string): Promise<void>;
}

interface TriageQueue {
  facilityId: string;
  cases: PatientCase[];
  lastUpdated: Date;
  statistics: QueueStatistics;
}

interface PatientCase {
  caseId: string;
  patientId: string;
  patientName: string;
  age: number;
  gender: Gender;
  riskLevel: RiskLevel;
  urgencyScore: number;
  primarySymptoms: string[];
  arrivalTime: Date;
  waitTime: number; // minutes
  status: CaseStatus;
  assignedDoctor?: string;
}

enum CaseStatus {
  PENDING = 'pending',
  IN_PROGRESS = 'in_progress',
  COMPLETED = 'completed',
  ESCALATED = 'escalated',
  CANCELLED = 'cancelled'
}

interface QueueStatistics {
  totalCases: number;
  highRiskCount: number;
  moderateRiskCount: number;
  lowRiskCount: number;
  averageWaitTime: number;
  longestWaitTime: number;
}
```

**4.2 SOAP Note Generator**
```typescript
interface SOAPNoteGenerator {
  generateSOAPNote(consultation: Consultation): Promise<SOAPNote>;
  updateSOAPNote(noteId: string, updates: Partial<SOAPNote>): Promise<SOAPNote>;
  approveSOAPNote(noteId: string, doctorId: string): Promise<void>;
  getSOAPHistory(patientId: string): Promise<SOAPNote[]>;
}

interface SOAPNote {
  id: string;
  patientId: string;
  consultationId: string;
  createdAt: Date;
  createdBy: string; // doctor ID
  status: NoteStatus;
  
  subjective: SubjectiveSection;
  objective: ObjectiveSection;
  assessment: AssessmentSection;
  plan: PlanSection;
  
  aiGenerated: boolean;
  doctorModified: boolean;
  originalAIVersion?: SOAPNote; // for audit trail
}

interface SubjectiveSection {
  chiefComplaint: string;
  historyOfPresentIllness: string;
  symptoms: Symptom[];
  patientNarrative: string;
  reviewOfSystems?: string;
  aiGenerated: boolean;
}

interface ObjectiveSection {
  vitalSigns: VitalSigns;
  physicalExamination: ExaminationFindings;
  labResults?: LabResult[];
  imagingResults?: ImagingResult[];
  aiGenerated: boolean;
}

interface AssessmentSection {
  differentialDiagnoses: Diagnosis[];
  workingDiagnosis?: Diagnosis;
  clinicalImpression: string;
  aiGenerated: boolean;
  aiConfidence?: number;
}

interface PlanSection {
  medications: Medication[];
  investigations: Investigation[];
  procedures?: Procedure[];
  referrals?: Referral[];
  followUp: FollowUpPlan;
  patientEducation: string[];
  aiGenerated: boolean;
  aiConfidence?: number;
}

enum NoteStatus {
  DRAFT = 'draft',
  AI_GENERATED = 'ai_generated',
  DOCTOR_REVIEW = 'doctor_review',
  APPROVED = 'approved',
  FINALIZED = 'finalized'
}
```

**4.3 Clinical Suggestion Engine**
```typescript
interface ClinicalSuggestionEngine {
  generateDifferentialDiagnoses(case: PatientCase): Promise<DiagnosisSuggestion[]>;
  suggestTreatmentOptions(diagnosis: Diagnosis, patient: Patient): Promise<TreatmentSuggestion[]>;
  suggestInvestigations(symptoms: Symptom[], history: MedicalHistory): Promise<InvestigationSuggestion[]>;
  explainSuggestion(suggestionId: string): Promise<SuggestionExplanation>;
}

interface DiagnosisSuggestion {
  id: string;
  diagnosis: Diagnosis;
  probability: number;
  confidence: number;
  supportingEvidence: Evidence[];
  contradictingEvidence: Evidence[];
  guidelines: MedicalGuideline[];
  requiresDoctorReview: boolean;
}

interface TreatmentSuggestion {
  id: string;
  type: TreatmentType; // medication, procedure, lifestyle
  description: string;
  medications?: Medication[];
  dosageRecommendation?: string;
  duration?: string;
  contraindications: string[];
  sideEffects: string[];
  evidence: Evidence[];
  confidence: number;
}

interface Evidence {
  type: EvidenceType; // symptom, lab_result, history, guideline
  description: string;
  weight: number; // importance 0-1
  source?: string;
}

enum EvidenceType {
  SYMPTOM = 'symptom',
  LAB_RESULT = 'lab_result',
  VITAL_SIGN = 'vital_sign',
  MEDICAL_HISTORY = 'medical_history',
  PHYSICAL_EXAM = 'physical_exam',
  GUIDELINE = 'guideline',
  RESEARCH = 'research'
}
```


**4.4 Patient History Manager**
```typescript
interface PatientHistoryManager {
  getLongitudinalHistory(patientId: string, filters?: HistoryFilters): Promise<PatientHistory>;
  searchHistory(patientId: string, query: HistoryQuery): Promise<HistorySearchResult[]>;
  getTimelineVisualization(patientId: string): Promise<HealthTimeline>;
  aggregateFromMultipleFacilities(patientId: string): Promise<UnifiedHistory>;
}

interface PatientHistory {
  patientId: string;
  visits: Visit[];
  diagnoses: DiagnosisHistory[];
  medications: MedicationHistory[];
  labResults: LabResult[];
  procedures: Procedure[];
  allergies: Allergy[];
  chronicConditions: ChronicCondition[];
  immunizations: Immunization[];
}

interface Visit {
  visitId: string;
  date: Date;
  facilityId: string;
  facilityName: string;
  doctorId: string;
  doctorName: string;
  chiefComplaint: string;
  diagnosis: string[];
  prescriptions: Medication[];
  labOrders: Investigation[];
  notes: string;
}

interface HealthTimeline {
  patientId: string;
  events: TimelineEvent[];
  milestones: HealthMilestone[];
  trends: HealthTrend[];
}

interface TimelineEvent {
  date: Date;
  type: EventType; // visit, diagnosis, medication, lab, procedure
  title: string;
  description: string;
  severity?: SeverityLevel;
  facility?: string;
}

interface HealthTrend {
  metric: string; // blood_pressure, glucose, weight
  dataPoints: DataPoint[];
  trend: TrendDirection; // improving, stable, worsening
  analysis: string;
}
```

**4.5 OCR Engine**
```typescript
interface OCREngine {
  processLabReport(image: Buffer): Promise<LabReportExtraction>;
  extractStructuredData(extraction: LabReportExtraction): Promise<StructuredLabData>;
  validateExtraction(data: StructuredLabData): Promise<ValidationResult>;
  correctOCRErrors(data: StructuredLabData, corrections: Correction[]): Promise<StructuredLabData>;
}

interface LabReportExtraction {
  rawText: string;
  confidence: number;
  detectedFormat: LabReportFormat;
  extractedValues: ExtractedValue[];
  requiresManualReview: boolean;
}

interface ExtractedValue {
  testName: string;
  result: string;
  unit?: string;
  referenceRange?: string;
  flag?: AbnormalFlag; // high, low, critical
  confidence: number;
  boundingBox: BoundingBox;
}

interface StructuredLabData {
  reportId: string;
  patientId: string;
  reportDate: Date;
  labName: string;
  tests: LabTest[];
  summary: string;
  abnormalFindings: AbnormalFinding[];
}

interface LabTest {
  testName: string;
  standardCode?: string; // LOINC code
  result: number | string;
  unit: string;
  referenceRange: ReferenceRange;
  isAbnormal: boolean;
  flag?: AbnormalFlag;
  interpretation?: string;
}

enum AbnormalFlag {
  HIGH = 'high',
  LOW = 'low',
  CRITICAL_HIGH = 'critical_high',
  CRITICAL_LOW = 'critical_low',
  ABNORMAL = 'abnormal'
}
```

**4.6 Document QA System**
```typescript
interface DocumentQASystem {
  askQuestion(patientId: string, question: string, language: LanguageCode): Promise<QAResponse>;
  searchDocuments(patientId: string, query: string): Promise<DocumentSearchResult[]>;
  getRelevantContext(patientId: string, topic: string): Promise<ContextualInformation>;
}

interface QAResponse {
  question: string;
  answer: string;
  confidence: number;
  sources: DocumentSource[];
  relatedQuestions: string[];
  noAnswerFound: boolean;
}

interface DocumentSource {
  documentId: string;
  documentType: DocumentType;
  date: Date;
  excerpt: string;
  relevanceScore: number;
  pageNumber?: number;
}

enum DocumentType {
  SOAP_NOTE = 'soap_note',
  LAB_REPORT = 'lab_report',
  PRESCRIPTION = 'prescription',
  IMAGING_REPORT = 'imaging_report',
  DISCHARGE_SUMMARY = 'discharge_summary',
  REFERRAL_LETTER = 'referral_letter'
}
```


### 5. AI Service

**Responsibilities:**
- Interface with AWS Bedrock for AI model inference
- Manage prompt engineering and model selection
- Handle confidence scoring and uncertainty quantification
- Implement explainability and interpretability features

**Key Components:**

**5.1 AI Model Manager**
```typescript
interface AIModelManager {
  invokeModel(request: ModelRequest): Promise<ModelResponse>;
  selectModel(task: AITask): ModelConfig;
  monitorModelPerformance(modelId: string): Promise<PerformanceMetrics>;
  updateModel(modelId: string, version: string): Promise<void>;
}

interface ModelRequest {
  modelId: string;
  prompt: string;
  parameters: ModelParameters;
  context?: string[];
  maxTokens?: number;
  temperature?: number;
}

interface ModelResponse {
  output: string;
  confidence: number;
  tokensUsed: number;
  latency: number;
  modelVersion: string;
  metadata: Record<string, any>;
}

interface ModelConfig {
  modelId: string;
  provider: 'bedrock' | 'custom';
  modelName: string;
  version: string;
  capabilities: AICapability[];
  maxContextLength: number;
  costPerToken: number;
}

enum AITask {
  SYMPTOM_EXTRACTION = 'symptom_extraction',
  RISK_CLASSIFICATION = 'risk_classification',
  SOAP_GENERATION = 'soap_generation',
  DIFFERENTIAL_DIAGNOSIS = 'differential_diagnosis',
  TREATMENT_SUGGESTION = 'treatment_suggestion',
  DOCUMENT_QA = 'document_qa',
  TRANSLATION = 'translation'
}
```

**5.2 Explainability Engine**
```typescript
interface ExplainabilityEngine {
  explainRiskClassification(classification: RiskClassification): Promise<Explanation>;
  explainDiagnosis(diagnosis: DiagnosisSuggestion): Promise<Explanation>;
  generateFeatureAttribution(input: any, output: any): Promise<FeatureAttribution>;
  generateCounterfactual(input: any, output: any): Promise<Counterfactual>;
}

interface Explanation {
  summary: string;
  detailedReasoning: string[];
  featureImportance: FeatureImportance[];
  evidenceChain: Evidence[];
  confidence: number;
  visualizations?: ExplanationVisualization[];
}

interface FeatureImportance {
  feature: string;
  importance: number; // 0-1
  contribution: 'positive' | 'negative';
  description: string;
}

interface Counterfactual {
  originalInput: any;
  originalOutput: any;
  modifiedInput: any;
  modifiedOutput: any;
  changes: InputChange[];
  explanation: string;
}

interface ExplanationVisualization {
  type: 'bar_chart' | 'heatmap' | 'decision_tree' | 'attention_map';
  data: any;
  title: string;
  description: string;
}
```

**5.3 Confidence Gate**
```typescript
interface ConfidenceGate {
  evaluateConfidence(output: AIOutput): ConfidenceEvaluation;
  shouldEscalate(evaluation: ConfidenceEvaluation): boolean;
  routeToHuman(caseId: string, reason: EscalationReason): Promise<void>;
  trackConfidenceMetrics(): Promise<ConfidenceMetrics>;
}

interface ConfidenceEvaluation {
  overallConfidence: number;
  componentConfidences: Record<string, number>;
  uncertaintyType: UncertaintyType;
  escalationRequired: boolean;
  escalationReason?: EscalationReason;
}

enum UncertaintyType {
  ALEATORIC = 'aleatoric', // inherent data noise
  EPISTEMIC = 'epistemic', // model knowledge gaps
  BOTH = 'both'
}

enum EscalationReason {
  LOW_CONFIDENCE = 'low_confidence',
  HIGH_RISK = 'high_risk',
  CONTRADICTORY_EVIDENCE = 'contradictory_evidence',
  OUT_OF_DISTRIBUTION = 'out_of_distribution',
  SAFETY_CONCERN = 'safety_concern'
}

interface ConfidenceMetrics {
  averageConfidence: number;
  escalationRate: number;
  confidenceDistribution: Record<string, number>;
  accuracyByConfidenceBand: Record<string, number>;
}
```


### 6. Integration Service

**Responsibilities:**
- Integrate with ABDM for health record exchange
- Manage emergency escalation notifications
- Handle facility and provider management
- Coordinate with external systems

**Key Components:**

**6.1 ABDM Integration**
```typescript
interface ABDMIntegration {
  retrieveHealthRecords(healthId: string): Promise<ABDMHealthRecord>;
  uploadEncounterData(encounter: Encounter): Promise<ABDMUploadResult>;
  syncPatientData(patientId: string): Promise<SyncResult>;
  handleSyncFailure(syncId: string): Promise<void>;
  getABDMAuditLog(patientId: string): Promise<ABDMAuditEntry[]>;
}

interface ABDMHealthRecord {
  healthId: string;
  demographics: Demographics;
  encounters: ABDMEncounter[];
  prescriptions: ABDMPrescription[];
  diagnosticReports: ABDMDiagnosticReport[];
  immunizations: ABDMImmunization[];
}

interface ABDMUploadResult {
  success: boolean;
  transactionId: string;
  timestamp: Date;
  errors?: ABDMError[];
}

interface SyncResult {
  syncId: string;
  status: SyncStatus;
  recordsSynced: number;
  errors: SyncError[];
  retryScheduled?: Date;
}

enum SyncStatus {
  SUCCESS = 'success',
  PARTIAL_SUCCESS = 'partial_success',
  FAILED = 'failed',
  QUEUED = 'queued',
  IN_PROGRESS = 'in_progress'
}
```

**6.2 Emergency Escalation**
```typescript
interface EmergencyEscalation {
  triggerEmergency(case: PatientCase): Promise<EscalationResult>;
  notifyRecipients(escalation: EmergencyEscalation): Promise<NotificationResult[]>;
  trackAcknowledgment(escalationId: string): Promise<AcknowledgmentStatus>;
  escalateToNextLevel(escalationId: string): Promise<void>;
  getDashboard(): Promise<EmergencyDashboard>;
}

interface EscalationResult {
  escalationId: string;
  caseId: string;
  timestamp: Date;
  recipients: Recipient[];
  channels: NotificationChannel[];
  status: EscalationStatus;
}

interface Recipient {
  id: string;
  type: RecipientType; // phc, district_officer, emergency_services
  name: string;
  contact: ContactInfo;
  priority: number;
}

enum RecipientType {
  PHC = 'phc',
  DISTRICT_OFFICER = 'district_officer',
  EMERGENCY_SERVICES = 'emergency_services',
  SPECIALIST = 'specialist',
  AMBULANCE = 'ambulance'
}

interface NotificationResult {
  recipientId: string;
  channel: NotificationChannel;
  status: DeliveryStatus;
  timestamp: Date;
  acknowledged: boolean;
  acknowledgedAt?: Date;
}

enum NotificationChannel {
  SMS = 'sms',
  PHONE_CALL = 'phone_call',
  APP_NOTIFICATION = 'app_notification',
  EMAIL = 'email',
  WHATSAPP = 'whatsapp'
}

interface EmergencyDashboard {
  activeEmergencies: EmergencyCase[];
  statistics: EmergencyStatistics;
  responseTimeMetrics: ResponseTimeMetrics;
}

interface EmergencyCase {
  escalationId: string;
  patientInfo: PatientInfo;
  symptoms: string[];
  location: Location;
  status: EmergencyStatus;
  timeElapsed: number;
  acknowledgedBy?: string[];
}

enum EmergencyStatus {
  TRIGGERED = 'triggered',
  ACKNOWLEDGED = 'acknowledged',
  EN_ROUTE = 'en_route',
  ARRIVED = 'arrived',
  RESOLVED = 'resolved',
  ESCALATED = 'escalated'
}
```

**6.3 Facility Manager**
```typescript
interface FacilityManager {
  getFacilities(filters: FacilityFilters): Promise<HealthFacility[]>;
  updateFacility(facilityId: string, updates: Partial<HealthFacility>): Promise<void>;
  findNearestFacility(location: Location, criteria: FacilityCriteria): Promise<HealthFacility>;
  checkCapacity(facilityId: string): Promise<CapacityInfo>;
  getProviderSchedule(providerId: string): Promise<Schedule>;
}

interface HealthFacility {
  id: string;
  name: string;
  type: FacilityType;
  location: Location;
  contact: ContactInfo;
  operatingHours: OperatingHours;
  services: MedicalService[];
  specialties: Specialty[];
  capacity: CapacityInfo;
  currentLoad: number;
  providers: Provider[];
}

enum FacilityType {
  PHC = 'phc',
  CHC = 'chc', // Community Health Center
  DISTRICT_HOSPITAL = 'district_hospital',
  MEDICAL_COLLEGE = 'medical_college',
  PRIVATE_HOSPITAL = 'private_hospital',
  CLINIC = 'clinic'
}

interface CapacityInfo {
  totalBeds: number;
  availableBeds: number;
  emergencyBeds: number;
  icuBeds: number;
  currentPatients: number;
  maxCapacity: number;
  utilizationRate: number;
}

interface Provider {
  id: string;
  name: string;
  qualification: string;
  specialty: Specialty;
  licenseNumber: string;
  availability: Availability;
  languages: LanguageCode[];
}
```


### 7. Security and Compliance Service

**Responsibilities:**
- Manage authentication and authorization
- Handle data encryption and key management
- Maintain audit logs and compliance reporting
- Monitor for security threats and data breaches

**Key Components:**

**7.1 Authentication Manager**
```typescript
interface AuthenticationManager {
  authenticate(credentials: Credentials): Promise<AuthResult>;
  verifyMFA(userId: string, code: string): Promise<boolean>;
  refreshToken(refreshToken: string): Promise<TokenPair>;
  logout(userId: string, sessionId: string): Promise<void>;
  lockAccount(userId: string, reason: string): Promise<void>;
}

interface Credentials {
  type: AuthType;
  username?: string;
  password?: string;
  otp?: string;
  biometric?: BiometricData;
}

enum AuthType {
  PASSWORD = 'password',
  OTP = 'otp',
  BIOMETRIC = 'biometric',
  SSO = 'sso'
}

interface AuthResult {
  success: boolean;
  userId?: string;
  tokens?: TokenPair;
  mfaRequired?: boolean;
  error?: AuthError;
}

interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  tokenType: 'Bearer';
}
```

**7.2 Authorization Manager**
```typescript
interface AuthorizationManager {
  checkPermission(userId: string, resource: string, action: Action): Promise<boolean>;
  getUserRole(userId: string): Promise<Role>;
  assignRole(userId: string, role: Role): Promise<void>;
  getRolePermissions(role: Role): Promise<Permission[]>;
}

enum Role {
  PATIENT = 'patient',
  DOCTOR = 'doctor',
  NURSE = 'nurse',
  ADMIN = 'admin',
  DISTRICT_OFFICER = 'district_officer',
  COMPLIANCE_OFFICER = 'compliance_officer',
  SYSTEM_ADMIN = 'system_admin'
}

interface Permission {
  resource: string;
  actions: Action[];
  conditions?: AccessCondition[];
}

enum Action {
  READ = 'read',
  WRITE = 'write',
  UPDATE = 'update',
  DELETE = 'delete',
  APPROVE = 'approve',
  EXPORT = 'export'
}

interface AccessCondition {
  type: ConditionType;
  value: any;
}

enum ConditionType {
  FACILITY_MATCH = 'facility_match',
  TIME_RANGE = 'time_range',
  IP_WHITELIST = 'ip_whitelist',
  PATIENT_CONSENT = 'patient_consent'
}
```

**7.3 Encryption Manager**
```typescript
interface EncryptionManager {
  encryptData(data: string, context: EncryptionContext): Promise<EncryptedData>;
  decryptData(encrypted: EncryptedData, context: EncryptionContext): Promise<string>;
  rotateKeys(): Promise<KeyRotationResult>;
  encryptField(value: string, fieldType: FieldType): Promise<string>;
  decryptField(encrypted: string, fieldType: FieldType): Promise<string>;
}

interface EncryptedData {
  ciphertext: string;
  keyId: string;
  algorithm: string;
  iv: string;
  tag?: string;
}

interface EncryptionContext {
  purpose: string;
  userId?: string;
  resourceId?: string;
}

enum FieldType {
  PII = 'pii',
  PHI = 'phi',
  SENSITIVE = 'sensitive',
  STANDARD = 'standard'
}

interface KeyRotationResult {
  oldKeyId: string;
  newKeyId: string;
  rotatedAt: Date;
  recordsReEncrypted: number;
}
```

**7.4 Audit Logger**
```typescript
interface AuditLogger {
  logAccess(event: AccessEvent): Promise<void>;
  logDataModification(event: ModificationEvent): Promise<void>;
  logSecurityEvent(event: SecurityEvent): Promise<void>;
  queryAuditLog(query: AuditQuery): Promise<AuditEntry[]>;
  generateComplianceReport(period: DateRange): Promise<ComplianceReport>;
}

interface AccessEvent {
  timestamp: Date;
  userId: string;
  userRole: Role;
  resource: string;
  action: Action;
  patientId?: string;
  purpose: string;
  ipAddress: string;
  success: boolean;
}

interface ModificationEvent {
  timestamp: Date;
  userId: string;
  resource: string;
  action: Action;
  before?: any;
  after?: any;
  reason?: string;
}

interface SecurityEvent {
  timestamp: Date;
  type: SecurityEventType;
  severity: Severity;
  description: string;
  userId?: string;
  ipAddress?: string;
  metadata: Record<string, any>;
}

enum SecurityEventType {
  FAILED_LOGIN = 'failed_login',
  ACCOUNT_LOCKED = 'account_locked',
  UNAUTHORIZED_ACCESS = 'unauthorized_access',
  DATA_BREACH_ATTEMPT = 'data_breach_attempt',
  SUSPICIOUS_ACTIVITY = 'suspicious_activity',
  ENCRYPTION_FAILURE = 'encryption_failure'
}

interface AuditEntry {
  id: string;
  timestamp: Date;
  type: AuditType;
  event: AccessEvent | ModificationEvent | SecurityEvent;
  hash: string; // tamper-proof hash
}

enum AuditType {
  ACCESS = 'access',
  MODIFICATION = 'modification',
  SECURITY = 'security'
}
```


### 8. Monitoring and Analytics Service

**Responsibilities:**
- Monitor system performance and health
- Track AI model performance and bias
- Generate analytics and reporting dashboards
- Alert on anomalies and issues

**Key Components:**

**8.1 Performance Monitor**
```typescript
interface PerformanceMonitor {
  trackMetric(metric: Metric): Promise<void>;
  getMetrics(query: MetricQuery): Promise<MetricData[]>;
  setAlert(alert: AlertRule): Promise<void>;
  checkThresholds(): Promise<AlertTrigger[]>;
}

interface Metric {
  name: string;
  value: number;
  unit: string;
  timestamp: Date;
  tags: Record<string, string>;
}

interface MetricQuery {
  metricNames: string[];
  timeRange: DateRange;
  aggregation: AggregationType;
  groupBy?: string[];
  filters?: Record<string, any>;
}

enum AggregationType {
  AVG = 'avg',
  SUM = 'sum',
  MIN = 'min',
  MAX = 'max',
  COUNT = 'count',
  PERCENTILE = 'percentile'
}

interface AlertRule {
  id: string;
  name: string;
  metric: string;
  condition: AlertCondition;
  threshold: number;
  duration: number; // seconds
  severity: Severity;
  recipients: string[];
}

interface AlertCondition {
  operator: ComparisonOperator;
  value: number;
  aggregation: AggregationType;
}

enum ComparisonOperator {
  GREATER_THAN = 'gt',
  LESS_THAN = 'lt',
  EQUALS = 'eq',
  NOT_EQUALS = 'ne'
}
```

**8.2 Bias Monitor**
```typescript
interface BiasMonitor {
  trackAIOutput(output: AIOutput, demographics: Demographics): Promise<void>;
  generateFairnessReport(period: DateRange): Promise<FairnessReport>;
  detectBias(metric: string, threshold: number): Promise<BiasDetection[]>;
  trackOutcomes(outcomes: Outcome[]): Promise<void>;
}

interface FairnessReport {
  period: DateRange;
  metrics: FairnessMetric[];
  biasDetected: BiasDetection[];
  recommendations: string[];
  complianceStatus: ComplianceStatus;
}

interface FairnessMetric {
  name: string;
  overallValue: number;
  byDemographic: Record<string, number>;
  disparity: number; // max difference between groups
  threshold: number;
  passed: boolean;
}

interface BiasDetection {
  metric: string;
  affectedGroup: string;
  disparity: number;
  threshold: number;
  severity: Severity;
  detectedAt: Date;
  rootCause?: string;
}

interface Demographics {
  gender?: Gender;
  ageGroup?: AgeGroup;
  language?: LanguageCode;
  geography?: GeographyType;
  socioeconomicStatus?: SocioeconomicLevel;
}

enum GeographyType {
  URBAN = 'urban',
  RURAL = 'rural',
  SEMI_URBAN = 'semi_urban'
}

enum AgeGroup {
  YOUNG_ADULT = '18-30',
  ADULT = '31-50',
  SENIOR = '51-70',
  ELDERLY = '70+'
}
```

**8.3 Analytics Engine**
```typescript
interface AnalyticsEngine {
  generateDashboard(type: DashboardType, filters: DashboardFilters): Promise<Dashboard>;
  exportReport(reportType: ReportType, format: ExportFormat): Promise<Buffer>;
  predictTrends(metric: string, horizon: number): Promise<TrendPrediction>;
  identifyPatterns(data: DataSet): Promise<Pattern[]>;
}

interface Dashboard {
  type: DashboardType;
  widgets: Widget[];
  lastUpdated: Date;
  refreshInterval: number;
}

enum DashboardType {
  SYSTEM_HEALTH = 'system_health',
  CLINICAL_METRICS = 'clinical_metrics',
  EMERGENCY_OVERVIEW = 'emergency_overview',
  FACILITY_PERFORMANCE = 'facility_performance',
  DISEASE_SURVEILLANCE = 'disease_surveillance',
  BIAS_MONITORING = 'bias_monitoring'
}

interface Widget {
  id: string;
  type: WidgetType;
  title: string;
  data: any;
  visualization: VisualizationType;
}

enum WidgetType {
  METRIC = 'metric',
  CHART = 'chart',
  TABLE = 'table',
  MAP = 'map',
  ALERT = 'alert'
}

enum VisualizationType {
  LINE_CHART = 'line_chart',
  BAR_CHART = 'bar_chart',
  PIE_CHART = 'pie_chart',
  HEATMAP = 'heatmap',
  TIMELINE = 'timeline',
  GEOGRAPHIC_MAP = 'geographic_map'
}

interface TrendPrediction {
  metric: string;
  currentValue: number;
  predictions: PredictionPoint[];
  confidence: number;
  methodology: string;
}

interface PredictionPoint {
  timestamp: Date;
  predictedValue: number;
  confidenceInterval: [number, number];
}
```


## Data Models

### Core Entities

**Patient**
```typescript
interface Patient {
  id: string;
  abdmHealthId?: string;
  demographics: Demographics;
  contact: ContactInfo;
  emergencyContact: ContactInfo;
  preferredLanguage: LanguageCode;
  consent: ConsentRecord;
  createdAt: Date;
  updatedAt: Date;
}

interface Demographics {
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  age: number;
  gender: Gender;
  bloodGroup?: BloodGroup;
  maritalStatus?: MaritalStatus;
  occupation?: string;
  address: Address;
}

enum Gender {
  MALE = 'male',
  FEMALE = 'female',
  OTHER = 'other',
  PREFER_NOT_TO_SAY = 'prefer_not_to_say'
}

interface Address {
  line1: string;
  line2?: string;
  village?: string;
  district: string;
  state: string;
  pincode: string;
  country: string;
  coordinates?: Coordinates;
}

interface ContactInfo {
  phoneNumber: string;
  alternatePhone?: string;
  email?: string;
  whatsapp?: string;
}

interface ConsentRecord {
  dataCollection: boolean;
  dataSharing: boolean;
  aiProcessing: boolean;
  abdmSync: boolean;
  research: boolean;
  consentedAt: Date;
  expiresAt?: Date;
}
```

**Consultation**
```typescript
interface Consultation {
  id: string;
  patientId: string;
  facilityId: string;
  doctorId: string;
  type: ConsultationType;
  mode: ConsultationMode;
  startTime: Date;
  endTime?: Date;
  status: ConsultationStatus;
  
  chiefComplaint: string;
  symptoms: Symptom[];
  vitalSigns?: VitalSigns;
  examination?: ExaminationFindings;
  
  riskAssessment?: RiskClassification;
  soapNote?: SOAPNote;
  prescriptions: Medication[];
  investigations: Investigation[];
  
  followUp?: FollowUpPlan;
  referral?: Referral;
  
  aiAssisted: boolean;
  aiSuggestions?: AISuggestion[];
  
  createdAt: Date;
  updatedAt: Date;
}

enum ConsultationType {
  INITIAL = 'initial',
  FOLLOW_UP = 'follow_up',
  EMERGENCY = 'emergency',
  ROUTINE_CHECKUP = 'routine_checkup'
}

enum ConsultationMode {
  IN_PERSON = 'in_person',
  TELEMEDICINE = 'telemedicine',
  IVR_ASSESSMENT = 'ivr_assessment'
}

enum ConsultationStatus {
  SCHEDULED = 'scheduled',
  IN_PROGRESS = 'in_progress',
  COMPLETED = 'completed',
  CANCELLED = 'cancelled',
  NO_SHOW = 'no_show'
}

interface VitalSigns {
  temperature?: number; // Celsius
  bloodPressure?: BloodPressure;
  heartRate?: number; // bpm
  respiratoryRate?: number; // breaths per minute
  oxygenSaturation?: number; // percentage
  weight?: number; // kg
  height?: number; // cm
  bmi?: number;
  measuredAt: Date;
}

interface BloodPressure {
  systolic: number;
  diastolic: number;
}
```

**Medication**
```typescript
interface Medication {
  id: string;
  name: string;
  genericName?: string;
  dosage: string;
  route: MedicationRoute;
  frequency: string;
  duration: string;
  instructions: string;
  startDate: Date;
  endDate?: Date;
  prescribedBy: string;
  indication?: string;
  sideEffects?: string[];
  contraindications?: string[];
}

enum MedicationRoute {
  ORAL = 'oral',
  TOPICAL = 'topical',
  INJECTION = 'injection',
  INHALATION = 'inhalation',
  SUBLINGUAL = 'sublingual',
  RECTAL = 'rectal'
}
```

**Investigation**
```typescript
interface Investigation {
  id: string;
  type: InvestigationType;
  name: string;
  orderedBy: string;
  orderedAt: Date;
  status: InvestigationStatus;
  priority: Priority;
  indication: string;
  results?: InvestigationResult;
  reportUrl?: string;
}

enum InvestigationType {
  LAB_TEST = 'lab_test',
  IMAGING = 'imaging',
  BIOPSY = 'biopsy',
  ECG = 'ecg',
  ULTRASOUND = 'ultrasound',
  XRAY = 'xray',
  CT_SCAN = 'ct_scan',
  MRI = 'mri'
}

enum InvestigationStatus {
  ORDERED = 'ordered',
  SAMPLE_COLLECTED = 'sample_collected',
  IN_PROGRESS = 'in_progress',
  COMPLETED = 'completed',
  CANCELLED = 'cancelled'
}

interface InvestigationResult {
  completedAt: Date;
  findings: string;
  interpretation: string;
  abnormalFindings: AbnormalFinding[];
  reportedBy: string;
}

interface AbnormalFinding {
  parameter: string;
  value: string;
  normalRange: string;
  severity: Severity;
  clinicalSignificance: string;
}
```

**Diagnosis**
```typescript
interface Diagnosis {
  id: string;
  code: string; // ICD-10 code
  name: string;
  type: DiagnosisType;
  status: DiagnosisStatus;
  diagnosedAt: Date;
  diagnosedBy: string;
  confidence?: number;
  notes?: string;
}

enum DiagnosisType {
  PRIMARY = 'primary',
  SECONDARY = 'secondary',
  DIFFERENTIAL = 'differential',
  PROVISIONAL = 'provisional',
  CONFIRMED = 'confirmed'
}

enum DiagnosisStatus {
  ACTIVE = 'active',
  RESOLVED = 'resolved',
  CHRONIC = 'chronic',
  RULED_OUT = 'ruled_out'
}
```


## Database Schema

### PostgreSQL Tables

**patients**
```sql
CREATE TABLE patients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  abdm_health_id VARCHAR(50) UNIQUE,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  date_of_birth DATE NOT NULL,
  gender VARCHAR(20) NOT NULL,
  blood_group VARCHAR(5),
  phone_number VARCHAR(20) NOT NULL,
  email VARCHAR(255),
  preferred_language VARCHAR(5) NOT NULL,
  address JSONB NOT NULL,
  emergency_contact JSONB,
  consent JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP
);

CREATE INDEX idx_patients_abdm_health_id ON patients(abdm_health_id);
CREATE INDEX idx_patients_phone_number ON patients(phone_number);
CREATE INDEX idx_patients_created_at ON patients(created_at);
```

**consultations**
```sql
CREATE TABLE consultations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id UUID NOT NULL REFERENCES patients(id),
  facility_id UUID NOT NULL REFERENCES facilities(id),
  doctor_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  mode VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP,
  chief_complaint TEXT NOT NULL,
  symptoms JSONB,
  vital_signs JSONB,
  examination JSONB,
  risk_assessment JSONB,
  ai_assisted BOOLEAN DEFAULT false,
  ai_suggestions JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_consultations_patient_id ON consultations(patient_id);
CREATE INDEX idx_consultations_facility_id ON consultations(facility_id);
CREATE INDEX idx_consultations_doctor_id ON consultations(doctor_id);
CREATE INDEX idx_consultations_start_time ON consultations(start_time);
CREATE INDEX idx_consultations_status ON consultations(status);
```

**soap_notes**
```sql
CREATE TABLE soap_notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consultation_id UUID NOT NULL REFERENCES consultations(id),
  patient_id UUID NOT NULL REFERENCES patients(id),
  created_by UUID NOT NULL REFERENCES users(id),
  status VARCHAR(50) NOT NULL,
  subjective JSONB NOT NULL,
  objective JSONB NOT NULL,
  assessment JSONB NOT NULL,
  plan JSONB NOT NULL,
  ai_generated BOOLEAN DEFAULT false,
  doctor_modified BOOLEAN DEFAULT false,
  original_ai_version JSONB,
  approved_at TIMESTAMP,
  approved_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_soap_notes_consultation_id ON soap_notes(consultation_id);
CREATE INDEX idx_soap_notes_patient_id ON soap_notes(patient_id);
CREATE INDEX idx_soap_notes_created_by ON soap_notes(created_by);
```

**medications**
```sql
CREATE TABLE medications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consultation_id UUID NOT NULL REFERENCES consultations(id),
  patient_id UUID NOT NULL REFERENCES patients(id),
  name VARCHAR(255) NOT NULL,
  generic_name VARCHAR(255),
  dosage VARCHAR(100) NOT NULL,
  route VARCHAR(50) NOT NULL,
  frequency VARCHAR(100) NOT NULL,
  duration VARCHAR(100) NOT NULL,
  instructions TEXT,
  start_date DATE NOT NULL,
  end_date DATE,
  prescribed_by UUID NOT NULL REFERENCES users(id),
  indication TEXT,
  side_effects JSONB,
  contraindications JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_medications_patient_id ON medications(patient_id);
CREATE INDEX idx_medications_consultation_id ON medications(consultation_id);
CREATE INDEX idx_medications_start_date ON medications(start_date);
```

**investigations**
```sql
CREATE TABLE investigations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consultation_id UUID NOT NULL REFERENCES consultations(id),
  patient_id UUID NOT NULL REFERENCES patients(id),
  type VARCHAR(50) NOT NULL,
  name VARCHAR(255) NOT NULL,
  ordered_by UUID NOT NULL REFERENCES users(id),
  ordered_at TIMESTAMP NOT NULL,
  status VARCHAR(50) NOT NULL,
  priority VARCHAR(20) NOT NULL,
  indication TEXT,
  results JSONB,
  report_url TEXT,
  completed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_investigations_patient_id ON investigations(patient_id);
CREATE INDEX idx_investigations_consultation_id ON investigations(consultation_id);
CREATE INDEX idx_investigations_status ON investigations(status);
CREATE INDEX idx_investigations_ordered_at ON investigations(ordered_at);
```

**diagnoses**
```sql
CREATE TABLE diagnoses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consultation_id UUID NOT NULL REFERENCES consultations(id),
  patient_id UUID NOT NULL REFERENCES patients(id),
  code VARCHAR(20) NOT NULL,
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL,
  diagnosed_at TIMESTAMP NOT NULL,
  diagnosed_by UUID NOT NULL REFERENCES users(id),
  confidence DECIMAL(3,2),
  notes TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_diagnoses_patient_id ON diagnoses(patient_id);
CREATE INDEX idx_diagnoses_consultation_id ON diagnoses(consultation_id);
CREATE INDEX idx_diagnoses_code ON diagnoses(code);
CREATE INDEX idx_diagnoses_status ON diagnoses(status);
```

**users**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(100) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(50) NOT NULL,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  phone_number VARCHAR(20),
  facility_id UUID REFERENCES facilities(id),
  qualification VARCHAR(255),
  specialty VARCHAR(100),
  license_number VARCHAR(100),
  languages JSONB,
  mfa_enabled BOOLEAN DEFAULT false,
  mfa_secret VARCHAR(255),
  last_login TIMESTAMP,
  account_locked BOOLEAN DEFAULT false,
  locked_at TIMESTAMP,
  password_changed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_facility_id ON users(facility_id);
```

**facilities**
```sql
CREATE TABLE facilities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50) NOT NULL,
  location JSONB NOT NULL,
  contact JSONB NOT NULL,
  operating_hours JSONB NOT NULL,
  services JSONB,
  specialties JSONB,
  capacity JSONB,
  current_load INTEGER DEFAULT 0,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_facilities_type ON facilities(type);
CREATE INDEX idx_facilities_active ON facilities(active);
```

**audit_logs**
```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
  type VARCHAR(50) NOT NULL,
  user_id UUID REFERENCES users(id),
  user_role VARCHAR(50),
  resource VARCHAR(255) NOT NULL,
  action VARCHAR(50) NOT NULL,
  patient_id UUID REFERENCES patients(id),
  purpose TEXT,
  ip_address INET,
  success BOOLEAN NOT NULL,
  before_data JSONB,
  after_data JSONB,
  metadata JSONB,
  hash VARCHAR(64) NOT NULL
);

CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_patient_id ON audit_logs(patient_id);
CREATE INDEX idx_audit_logs_type ON audit_logs(type);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource);
```


**call_sessions**
```sql
CREATE TABLE call_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id VARCHAR(100) UNIQUE NOT NULL,
  patient_id UUID REFERENCES patients(id),
  phone_number VARCHAR(20) NOT NULL,
  language VARCHAR(5) NOT NULL,
  state VARCHAR(50) NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP,
  duration INTEGER,
  symptoms JSONB,
  conversation_history JSONB,
  risk_classification JSONB,
  audio_recordings JSONB,
  call_quality_metrics JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_call_sessions_session_id ON call_sessions(session_id);
CREATE INDEX idx_call_sessions_patient_id ON call_sessions(patient_id);
CREATE INDEX idx_call_sessions_phone_number ON call_sessions(phone_number);
CREATE INDEX idx_call_sessions_start_time ON call_sessions(start_time);
```

**emergency_escalations**
```sql
CREATE TABLE emergency_escalations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  escalation_id VARCHAR(100) UNIQUE NOT NULL,
  case_id UUID NOT NULL,
  patient_id UUID NOT NULL REFERENCES patients(id),
  consultation_id UUID REFERENCES consultations(id),
  status VARCHAR(50) NOT NULL,
  triggered_at TIMESTAMP NOT NULL,
  recipients JSONB NOT NULL,
  notifications JSONB,
  acknowledgments JSONB,
  resolved_at TIMESTAMP,
  resolution_notes TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_emergency_escalations_escalation_id ON emergency_escalations(escalation_id);
CREATE INDEX idx_emergency_escalations_patient_id ON emergency_escalations(patient_id);
CREATE INDEX idx_emergency_escalations_status ON emergency_escalations(status);
CREATE INDEX idx_emergency_escalations_triggered_at ON emergency_escalations(triggered_at);
```

**abdm_sync_logs**
```sql
CREATE TABLE abdm_sync_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sync_id VARCHAR(100) UNIQUE NOT NULL,
  patient_id UUID NOT NULL REFERENCES patients(id),
  consultation_id UUID REFERENCES consultations(id),
  operation VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL,
  transaction_id VARCHAR(100),
  records_synced INTEGER DEFAULT 0,
  errors JSONB,
  retry_count INTEGER DEFAULT 0,
  next_retry_at TIMESTAMP,
  synced_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_abdm_sync_logs_sync_id ON abdm_sync_logs(sync_id);
CREATE INDEX idx_abdm_sync_logs_patient_id ON abdm_sync_logs(patient_id);
CREATE INDEX idx_abdm_sync_logs_status ON abdm_sync_logs(status);
CREATE INDEX idx_abdm_sync_logs_created_at ON abdm_sync_logs(created_at);
```

### MongoDB Collections

**medical_documents**
```javascript
{
  _id: ObjectId,
  patientId: UUID,
  consultationId: UUID,
  type: String, // 'lab_report', 'prescription', 'imaging_report', etc.
  title: String,
  content: String, // full text content
  metadata: {
    uploadedBy: UUID,
    uploadedAt: Date,
    fileUrl: String,
    fileSize: Number,
    mimeType: String,
    ocrProcessed: Boolean,
    ocrConfidence: Number
  },
  structuredData: Object, // extracted structured data
  embeddings: [Number], // vector embeddings for semantic search
  tags: [String],
  createdAt: Date,
  updatedAt: Date
}

// Indexes
db.medical_documents.createIndex({ patientId: 1, createdAt: -1 });
db.medical_documents.createIndex({ consultationId: 1 });
db.medical_documents.createIndex({ type: 1 });
db.medical_documents.createIndex({ "metadata.uploadedAt": -1 });
```

**ai_model_outputs**
```javascript
{
  _id: ObjectId,
  modelId: String,
  modelVersion: String,
  task: String, // 'risk_classification', 'soap_generation', etc.
  input: Object,
  output: Object,
  confidence: Number,
  explainability: {
    featureImportance: [Object],
    reasoning: [String],
    evidenceChain: [Object]
  },
  metadata: {
    userId: UUID,
    patientId: UUID,
    consultationId: UUID,
    sessionId: String,
    latency: Number,
    tokensUsed: Number
  },
  safetyChecks: {
    guardrailsPassed: Boolean,
    blockedContent: [Object],
    interventions: [Object]
  },
  humanReview: {
    required: Boolean,
    reviewed: Boolean,
    reviewedBy: UUID,
    reviewedAt: Date,
    feedback: String,
    accepted: Boolean
  },
  createdAt: Date
}

// Indexes
db.ai_model_outputs.createIndex({ modelId: 1, createdAt: -1 });
db.ai_model_outputs.createIndex({ task: 1 });
db.ai_model_outputs.createIndex({ "metadata.patientId": 1 });
db.ai_model_outputs.createIndex({ "humanReview.required": 1, "humanReview.reviewed": 1 });
db.ai_model_outputs.createIndex({ createdAt: -1 });
```

**bias_monitoring_data**
```javascript
{
  _id: ObjectId,
  timestamp: Date,
  modelId: String,
  task: String,
  output: Object,
  demographics: {
    gender: String,
    ageGroup: String,
    language: String,
    geography: String,
    socioeconomicStatus: String
  },
  outcome: {
    riskLevel: String,
    escalated: Boolean,
    emergencyTriggered: Boolean,
    confidence: Number
  },
  metadata: {
    patientId: UUID,
    consultationId: UUID,
    facilityId: UUID
  }
}

// Indexes
db.bias_monitoring_data.createIndex({ timestamp: -1 });
db.bias_monitoring_data.createIndex({ modelId: 1, task: 1 });
db.bias_monitoring_data.createIndex({ "demographics.gender": 1 });
db.bias_monitoring_data.createIndex({ "demographics.ageGroup": 1 });
db.bias_monitoring_data.createIndex({ "demographics.language": 1 });
db.bias_monitoring_data.createIndex({ "demographics.geography": 1 });
```


## API Specifications

### REST API Endpoints (Mode A - Patient Service)

**IVR Call Management**

```
POST /api/v1/ivr/calls
Description: Initiate a new IVR call session
Request Body:
{
  "phoneNumber": "string",
  "callId": "string"
}
Response: 200 OK
{
  "sessionId": "string",
  "status": "initiated",
  "timestamp": "ISO8601"
}
```

```
POST /api/v1/ivr/calls/{sessionId}/voice-input
Description: Process voice input from patient
Request Body:
{
  "audioData": "base64",
  "language": "string"
}
Response: 200 OK
{
  "transcription": "string",
  "confidence": 0.95,
  "response": {
    "text": "string",
    "audioUrl": "string"
  },
  "nextAction": "continue" | "end"
}
```

```
POST /api/v1/ivr/calls/{sessionId}/dtmf-input
Description: Process DTMF keypad input
Request Body:
{
  "digits": "string"
}
Response: 200 OK
{
  "interpretation": "string",
  "response": {
    "text": "string",
    "audioUrl": "string"
  }
}
```

```
POST /api/v1/ivr/calls/{sessionId}/language
Description: Set language for call session
Request Body:
{
  "languageCode": "string"
}
Response: 200 OK
{
  "language": "string",
  "confirmed": true
}
```

```
POST /api/v1/ivr/calls/{sessionId}/end
Description: End call session
Request Body:
{
  "reason": "completed" | "disconnected" | "error"
}
Response: 200 OK
{
  "summary": {
    "duration": 180,
    "symptomsCollected": 5,
    "riskLevel": "moderate",
    "recommendation": "string"
  }
}
```

**Risk Assessment**

```
POST /api/v1/assessment/classify-risk
Description: Classify patient risk level
Request Body:
{
  "patientId": "string",
  "symptoms": [
    {
      "name": "string",
      "severity": 7,
      "duration": "2 days",
      "onset": "sudden"
    }
  ],
  "demographics": {
    "age": 45,
    "gender": "male"
  },
  "medicalHistory": {}
}
Response: 200 OK
{
  "riskLevel": "high",
  "confidence": 0.87,
  "urgencyScore": 85,
  "reasoning": ["string"],
  "recommendedAction": {
    "type": "emergency",
    "timeframe": "immediately",
    "instructions": ["string"]
  },
  "requiresHumanReview": false
}
```

### GraphQL API (Mode B - Clinical Service)

**Schema Definition**

```graphql
type Query {
  # Triage Queue
  triageQueue(facilityId: ID!, filters: QueueFilters): TriageQueue!
  patientCase(caseId: ID!): PatientCase!
  
  # Patient Management
  patient(id: ID!): Patient!
  patientHistory(patientId: ID!, filters: HistoryFilters): PatientHistory!
  searchPatients(query: String!, limit: Int): [Patient!]!
  
  # Consultations
  consultation(id: ID!): Consultation!
  consultations(patientId: ID!, limit: Int): [Consultation!]!
  
  # SOAP Notes
  soapNote(id: ID!): SOAPNote!
  soapNotes(patientId: ID!): [SOAPNote!]!
  
  # Documents
  medicalDocuments(patientId: ID!, type: DocumentType): [MedicalDocument!]!
  documentQA(patientId: ID!, question: String!, language: LanguageCode!): QAResponse!
  
  # Facilities
  facilities(filters: FacilityFilters): [Facility!]!
  nearestFacility(location: LocationInput!, criteria: FacilityCriteria!): Facility
  
  # Analytics
  dashboard(type: DashboardType!, filters: DashboardFilters): Dashboard!
  metrics(query: MetricQuery!): [MetricData!]!
}

type Mutation {
  # Patient Management
  createPatient(input: CreatePatientInput!): Patient!
  updatePatient(id: ID!, input: UpdatePatientInput!): Patient!
  
  # Consultations
  startConsultation(input: StartConsultationInput!): Consultation!
  updateConsultation(id: ID!, input: UpdateConsultationInput!): Consultation!
  completeConsultation(id: ID!): Consultation!
  
  # SOAP Notes
  generateSOAPNote(consultationId: ID!): SOAPNote!
  updateSOAPNote(id: ID!, input: UpdateSOAPNoteInput!): SOAPNote!
  approveSOAPNote(id: ID!): SOAPNote!
  
  # Prescriptions
  addMedication(consultationId: ID!, input: MedicationInput!): Medication!
  updateMedication(id: ID!, input: MedicationInput!): Medication!
  
  # Investigations
  orderInvestigation(consultationId: ID!, input: InvestigationInput!): Investigation!
  updateInvestigationResults(id: ID!, results: InvestigationResultInput!): Investigation!
  
  # Documents
  uploadDocument(input: UploadDocumentInput!): MedicalDocument!
  processLabReport(documentId: ID!): LabReportExtraction!
  
  # AI Suggestions
  requestClinicalSuggestions(consultationId: ID!): ClinicalSuggestions!
  acceptSuggestion(suggestionId: ID!): Boolean!
  rejectSuggestion(suggestionId: ID!, reason: String!): Boolean!
  
  # Emergency
  triggerEmergency(caseId: ID!): EmergencyEscalation!
  acknowledgeEmergency(escalationId: ID!, userId: ID!): Boolean!
}

type Subscription {
  # Real-time updates
  triageQueueUpdated(facilityId: ID!): TriageQueue!
  newHighRiskCase(facilityId: ID!): PatientCase!
  emergencyTriggered(facilityId: ID!): EmergencyEscalation!
  consultationUpdated(consultationId: ID!): Consultation!
}

# Types
type TriageQueue {
  facilityId: ID!
  cases: [PatientCase!]!
  statistics: QueueStatistics!
  lastUpdated: DateTime!
}

type PatientCase {
  caseId: ID!
  patient: Patient!
  riskLevel: RiskLevel!
  urgencyScore: Int!
  primarySymptoms: [String!]!
  arrivalTime: DateTime!
  waitTime: Int!
  status: CaseStatus!
  assignedDoctor: User
}

type Patient {
  id: ID!
  abdmHealthId: String
  demographics: Demographics!
  contact: ContactInfo!
  preferredLanguage: LanguageCode!
  consent: ConsentRecord!
  history: PatientHistory
}

type PatientHistory {
  visits: [Visit!]!
  diagnoses: [DiagnosisHistory!]!
  medications: [MedicationHistory!]!
  labResults: [LabResult!]!
  timeline: HealthTimeline!
}

type Consultation {
  id: ID!
  patient: Patient!
  facility: Facility!
  doctor: User!
  type: ConsultationType!
  status: ConsultationStatus!
  startTime: DateTime!
  endTime: DateTime
  chiefComplaint: String!
  symptoms: [Symptom!]!
  vitalSigns: VitalSigns
  riskAssessment: RiskClassification
  soapNote: SOAPNote
  prescriptions: [Medication!]!
  investigations: [Investigation!]!
  aiAssisted: Boolean!
}

type SOAPNote {
  id: ID!
  consultation: Consultation!
  status: NoteStatus!
  subjective: SubjectiveSection!
  objective: ObjectiveSection!
  assessment: AssessmentSection!
  plan: PlanSection!
  aiGenerated: Boolean!
  doctorModified: Boolean!
  approvedAt: DateTime
  approvedBy: User
}

type ClinicalSuggestions {
  differentialDiagnoses: [DiagnosisSuggestion!]!
  treatmentOptions: [TreatmentSuggestion!]!
  investigations: [InvestigationSuggestion!]!
}

# Enums
enum RiskLevel {
  LOW
  MODERATE
  HIGH
}

enum CaseStatus {
  PENDING
  IN_PROGRESS
  COMPLETED
  ESCALATED
  CANCELLED
}

enum ConsultationType {
  INITIAL
  FOLLOW_UP
  EMERGENCY
  ROUTINE_CHECKUP
}

enum ConsultationStatus {
  SCHEDULED
  IN_PROGRESS
  COMPLETED
  CANCELLED
  NO_SHOW
}

enum NoteStatus {
  DRAFT
  AI_GENERATED
  DOCTOR_REVIEW
  APPROVED
  FINALIZED
}

enum LanguageCode {
  EN
  HI
  BN
  TE
  MR
  TA
  GU
  KN
  ML
  OR
  PA
}
```


## Security Architecture

### 5-Layer Safety Framework

**Layer 1: Input Validation**
- Sanitize all user inputs to prevent injection attacks
- Validate data types, formats, and ranges
- Implement rate limiting to prevent abuse
- Detect and block malicious patterns

**Layer 2: AI Safety Guardrails**
- Block diagnoses, medications, and dosages in patient-facing outputs
- Implement content filtering using keyword matching and NLP
- Maintain prohibited content database with medical terminology
- Log all guardrail interventions for audit

**Layer 3: Confidence Gating**
- Require human review when AI confidence <60%
- Escalate contradictory or uncertain cases
- Track confidence distributions and accuracy by confidence band
- Implement uncertainty quantification (aleatoric + epistemic)

**Layer 4: Human-in-the-Loop**
- Mandatory doctor approval for all clinical decisions
- Allow doctors to override AI suggestions with documented reasoning
- Collect human feedback for continuous model improvement
- Maintain audit trail of AI suggestions vs. final decisions

**Layer 5: Post-Deployment Monitoring**
- Continuous monitoring of AI outputs for safety issues
- Bias monitoring across demographic dimensions
- Track adverse events and near-misses
- Implement automated alerts for anomalies

### Authentication Flow

```
1. User Login Request
   ↓
2. Validate Credentials (password hash comparison)
   ↓
3. Check Account Status (not locked, active)
   ↓
4. MFA Challenge (if enabled)
   ↓
5. Generate JWT Access Token (15 min expiry)
   ↓
6. Generate Refresh Token (7 day expiry)
   ↓
7. Store Session in Redis
   ↓
8. Return Tokens to Client
   ↓
9. Client includes Access Token in Authorization header
   ↓
10. API Gateway validates token signature and expiry
    ↓
11. Extract user ID and role from token claims
    ↓
12. Check permissions for requested resource/action
    ↓
13. Allow or deny request
```

### Data Encryption Strategy

**At Rest:**
- Database: AES-256-GCM encryption at column level for PII/PHI
- Object Storage: Server-side encryption with AWS KMS
- Backups: Separate encryption keys stored in geographically distributed HSMs
- Key Rotation: Automated 90-day rotation with re-encryption

**In Transit:**
- TLS 1.3 with perfect forward secrecy
- Certificate pinning for mobile apps
- Mutual TLS for service-to-service communication
- VPN for admin access

**Key Management:**
- AWS KMS with HSM backing for master keys
- Envelope encryption for data encryption keys
- Separate keys for different data classifications
- Key access logging and monitoring

### Access Control Matrix

| Role | Patient Data | Clinical Notes | Prescriptions | AI Suggestions | System Config | Audit Logs |
|------|--------------|----------------|---------------|----------------|---------------|------------|
| Patient | Own only (R) | Own only (R) | Own only (R) | No | No | Own only (R) |
| Doctor | Assigned (RW) | Create/Edit (RW) | Create/Edit (RW) | View/Accept (RW) | No | Own actions (R) |
| Nurse | Assigned (R) | View only (R) | View only (R) | No | No | No |
| Admin | All (R) | All (R) | All (R) | No | Facility (RW) | Facility (R) |
| District Officer | District (R) | District (R) | District (R) | No | District (R) | District (R) |
| Compliance Officer | All (R) | All (R) | All (R) | All (R) | No | All (R) |
| System Admin | No | No | No | No | All (RW) | All (R) |

R = Read, W = Write


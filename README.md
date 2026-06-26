import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, getDoc, onSnapshot } from 'firebase/firestore';

// --- FIREBASE CONFIGURATION & INITIALIZATION ---
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'sfl-healthcare-sales-crm';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

let app, auth, db;
if (firebaseConfig) {
  try {
    app = initializeApp(firebaseConfig);
    auth = getAuth(app);
    db = getFirestore(app);
  } catch (e) {
    console.error("Firebase Init Error:", e);
  }
}

// --- ACTUAL DATA COMPILED FROM SOURCE CSV & PDF DOCUMENTS ---
const initialAccounts = [
  {
    id: "jackson",
    name: "Jackson Health System",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "J02400",
    address: "1611 NW 12th Ave, Miami, FL 33136",
    beds: 2400,
    zone: 4,
    pocs: [
      { name: "Carlos Mendez", title: "Chief Medical Officer", note: "Interests: Contract Expansion" },
      { name: "Patricia Nguyen", title: "Director of Nursing", note: "Interests: Supply Kits & Carts" },
      { name: "Daniel Rojas", title: "IT Director", note: "Interests: Telehealth Carts" },
      { name: "Daniel Cruz", title: "Telehealth Engineer", note: "Interests: Samsung Displays" },
      { name: "Raul Ordonez", title: "Vendor Relations / Compliance", note: "Interests: Vendor Consolidation" }
    ],
    products: {
      computing: "active",
      audiovisual: "pipeline",
      network: "active",
      security: "none",
      kiosks: "pipeline",
      software: "active",
      poc_carts: "active",
      telehealth: "pipeline",
      patient_exp: "active",
      infection_control: "active"
    }
  },
  {
    id: "uhealth",
    name: "UHealth University of Miami Health",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "U01149",
    address: "1475 NW 12th Avenue, Miami, FL 33136",
    beds: 560,
    zone: 4,
    pocs: [
      { name: "John Gomez", title: "IT Infrastructure Lead", note: "Discussing security protocols" },
      { name: "Michael Adams", title: "Clinical Support Specialist", note: "Barcode scanner assessment" },
      { name: "Sara Masty", title: "Director of Nursing Operations", note: "Interested in point-of-care workflows" }
    ],
    products: {
      computing: "pipeline",
      audiovisual: "none",
      network: "active",
      security: "pipeline",
      kiosks: "none",
      software: "active",
      poc_carts: "active",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "pipeline"
    }
  },
  {
    id: "baptist_miami",
    name: "Baptist Hospital of Miami",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "B00728",
    address: "8900 N Kendall Dr, Miami, FL 33176",
    beds: 728,
    zone: 4,
    pocs: [
      { name: "Connie Chan", title: "Director of Pharmacy", note: "LinkedIn connected; tracking cart rollout" }
    ],
    products: {
      computing: "active",
      audiovisual: "none",
      network: "active",
      security: "active",
      kiosks: "none",
      software: "pipeline",
      poc_carts: "active",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "active"
    }
  },
  {
    id: "memorial_regional",
    name: "Memorial Regional Hospital",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "M00703",
    address: "3501 Johnson St, Hollywood, FL 33021",
    beds: 703,
    zone: 3,
    pocs: [
      { name: "Sandra Torres", title: "Director of Surgical Services", note: "Q2 pricing trial pending" }
    ],
    products: {
      computing: "active",
      audiovisual: "none",
      network: "active",
      security: "none",
      kiosks: "none",
      software: "active",
      poc_carts: "pipeline",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "active"
    }
  },
  {
    id: "broward_imperial",
    name: "Broward Health Imperial Point",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "B00204",
    address: "6401 N Federal Hwy, Fort Lauderdale, FL 33308",
    beds: 204,
    zone: 3,
    pocs: [
      { name: "Steve Fredrickson", title: "Facilities Manager", note: "Front door check-in bottleneck observed" }
    ],
    products: {
      computing: "none",
      audiovisual: "none",
      network: "none",
      security: "none",
      kiosks: "pipeline",
      software: "none",
      poc_carts: "none",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  },
  {
    id: "holy_cross",
    name: "Holy Cross Hospital",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "H00571",
    address: "4725 N Federal Hwy, Fort Lauderdale, FL 33308",
    beds: 571,
    zone: 3,
    pocs: [
      { name: "Chris Greco", title: "Interim IT Director", note: "IT infrastructure modernization review" },
      { name: "Winson", title: "IT Front Desktop Support Specialist", note: "Provided Chris Greco introduction" }
    ],
    products: {
      computing: "pipeline",
      audiovisual: "none",
      network: "none",
      security: "none",
      kiosks: "none",
      software: "none",
      poc_carts: "none",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "pipeline"
    }
  },
  {
    id: "martin_memorial",
    name: "Martin Memorial Health Systems",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "M02165",
    address: "200 SE Hospital Avenue, Stuart, FL 34995",
    beds: 344,
    zone: 1,
    pocs: [
      { name: "Bruce Caldwell", title: "Purchasing Lead", note: "Standardizing on clinical monitors" }
    ],
    products: {
      computing: "active",
      audiovisual: "none",
      network: "active",
      security: "none",
      kiosks: "none",
      software: "none",
      poc_carts: "pipeline",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  },
  {
    id: "wellington_regional",
    name: "Wellington Regional Medical Center",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "W00159",
    address: "10100 Forest Hill Blvd, Wellington, FL 33414",
    beds: 159,
    zone: 2,
    pocs: [
      { name: "Keith Miller", title: "IT Team Lead", note: "Coordinating Alan B Miller facility handover" },
      { name: "Keith Pearson", title: "Facilities Contact", note: "Needs comprehensive on-site cart health check" }
    ],
    products: {
      computing: "active",
      audiovisual: "none",
      network: "none",
      security: "none",
      kiosks: "none",
      software: "none",
      poc_carts: "pipeline",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  },
  {
    id: "jupiter_med",
    name: "Jupiter Medical Center",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "J00163",
    address: "1210 S Old Dixie Hwy, Jupiter, FL 33458",
    beds: 163,
    zone: 1,
    pocs: [
      { name: "Kevin Olson", title: "Chief Information Officer", note: "IT gating. Direct outreach strategy required" },
      { name: "Fredric Simpson", title: "Facilities Specialist", note: "Historically locked in with legacy security provider" }
    ],
    products: {
      computing: "pipeline",
      audiovisual: "none",
      network: "none",
      security: "none",
      kiosks: "none",
      software: "none",
      poc_carts: "none",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  },
  {
    id: "pb_gardens",
    name: "Palm Beach Gardens Medical Center",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "P00204",
    address: "3360 Burns Rd, Palm Beach Gardens, FL 33410",
    beds: 204,
    zone: 2,
    pocs: [
      { name: "Sergio Valvez", title: "IT Manager (Contracted MSDL)", note: "Targeting Sophos licensing expirations" }
    ],
    products: {
      computing: "none",
      audiovisual: "none",
      network: "none",
      security: "pipeline",
      kiosks: "none",
      software: "none",
      poc_carts: "none",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  },
  {
    id: "jfk_med",
    name: "JFK Medical Center",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "J00424",
    address: "5301 S Congress Ave, Atlantis, FL 33462",
    beds: 424,
    zone: 2,
    pocs: [
      { name: "Beverly Lindsay", title: "Director of Operating Room", note: "Reviewing scanner refresh metrics" }
    ],
    products: {
      computing: "none",
      audiovisual: "none",
      network: "none",
      security: "none",
      kiosks: "none",
      software: "none",
      poc_carts: "none",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "pipeline"
    }
  },
  {
    id: "st_catherines",
    name: "St. Catherine's West Rehabilitation Hospital",
    type: "HEALTHCARE -- ACUTE CARE",
    number: "S00100",
    address: "8850 NW 122nd St, Hialeah, FL 33018",
    beds: 120,
    zone: 4,
    pocs: [
      { name: "Noslen Bacallao", title: "Director of Engineering & Security", note: "Physical safety project approved" }
    ],
    products: {
      computing: "none",
      audiovisual: "none",
      network: "none",
      security: "active",
      kiosks: "none",
      software: "none",
      poc_carts: "none",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  },
  {
    id: "west_palm_va",
    name: "VISN 8 - West Palm Beach VA",
    type: "HEALTHCARE -- VA",
    number: "V00318",
    address: "7305 N Military Trail, West Palm Beach, FL 33410",
    beds: 350,
    zone: 2,
    pocs: [
      { name: "Marie Jusma", title: "Lead Supervisor", note: "Direct procurement cycles" },
      { name: "Jeanette Travis", title: "Procurement Specialist", note: "Clinical computing & nursing tech rollout" },
      { name: "Michael Dixson", title: "IT Specialist", note: "Hardware refresh alignment" }
    ],
    products: {
      computing: "active",
      audiovisual: "active",
      network: "none",
      security: "none",
      kiosks: "none",
      software: "none",
      poc_carts: "active",
      telehealth: "none",
      patient_exp: "none",
      infection_control: "none"
    }
  }
];

const initialActivities = [
  {
    id: "act-1",
    date: "2026-05-20",
    account: "Memorial Regional Hospital",
    contact: "Sandra Torres",
    title: "Director of Surgical Services",
    type: "Meeting",
    notes: "On-site meeting with Director of Surgical Services; reviewed Q2 product line, specifications, and volume discounts. Very keen on specialized medical point-of-care supply carts.",
    outcome: "Positive",
    nextAction: "Send proposal for 3-month trial (by 05/23/2026)",
    beds: 703,
    zone: 3,
    source: "Healthcare Account & Contact Tracker"
  },
  {
    id: "act-2",
    date: "2026-05-19",
    account: "Jackson Health System",
    contact: "Carlos Mendez",
    title: "Chief Medical Officer",
    type: "Meeting",
    notes: "Lunch meeting with CMO; discussed expanding the current point-of-care tech contract to two additional floors (ICU & Pediatrics). Expressed strong interest in standardizing on our cart layouts.",
    outcome: "Positive",
    nextAction: "Draft expanded contract proposal (by 05/27/2026)",
    beds: 2400,
    zone: 4,
    source: "Healthcare Account & Contact Tracker"
  },
  {
    id: "act-3",
    date: "2026-05-19",
    account: "Jackson Health System",
    contact: "Patricia Nguyen",
    title: "Director of Nursing",
    type: "Demo",
    notes: "Product demo for nursing staff on our new clinical supply kit and barcode-integrated cart. 8 staff members attended. Engagement was very high, nursing leads appreciated the ergonomic benefits.",
    outcome: "Positive",
    nextAction: "Follow up with feedback survey results and pricing schedules (by 05/22/2026)",
    beds: 2400,
    zone: 4,
    source: "Healthcare Account & Contact Tracker"
  },
  {
    id: "act-4",
    date: "2026-05-19",
    account: "Internal — HHS CPG Program",
    contact: "Compliance Officers",
    title: "CCO / Privacy Officer / Procurement Lead",
    type: "Presentation",
    notes: "Delivered talk on HHS Cybersecurity Performance Goals (CPG) #10 regarding vendor cybersecurity and BAA compliance (HIPAA §164.308(b)). Audited 14 external supplier access lanes and flagged 6 legacy vendors failing cybersecurity addenda.",
    outcome: "Open",
    nextAction: "Issue supplier cure letters with Legal; response deadline set to 06/15/2026",
    beds: 0,
    zone: 3,
    source: "HHS CPG Cybersecurity Audit Log"
  },
  {
    id: "act-5",
    date: "2026-05-14",
    account: "Broward Health Imperial Point",
    contact: "Steve Fredrickson",
    title: "Facilities Manager",
    type: "Drop-By",
    notes: "Attempted to meet with Steve. Turned away due to heavy morning patient check-in traffic. Observed critical pain points in front lobby queue layout. Proposing lobby flow solution. Lou (Regional Lead) commented: 'Imperial is historically the forgotten facility for Broward, high opportunity area.'",
    outcome: "Needs Follow-Up",
    nextAction: "Consult with Kiosk design engineer to build lobby flow proposal; reach out to Lou for contact strategy",
    beds: 204,
    zone: 3,
    source: "CRM Q1 Meetings Log"
  },
  {
    id: "act-6",
    date: "2026-05-14",
    account: "Holy Cross Hospital",
    contact: "Chris Greco",
    title: "Interim IT Director",
    type: "Drop-By",
    notes: "Stopped by IT offices. Spoke with front desk technician Winson. He explained they have high vendor sales traffic, but Chris Greco manages IT infrastructure opportunities. Coordinated with Isiah (Code Corp partner) to see if we can bundle scanner promotions.",
    outcome: "Neutral",
    nextAction: "Email Chris Greco regarding IT infrastructure refresh; coordinate with Isiah on Code scanner resources",
    beds: 571,
    zone: 3,
    source: "CRM Q1 Meetings Log"
  },
  {
    id: "act-7",
    date: "2026-05-14",
    account: "Wellington Regional Medical Center",
    contact: "Keith Miller",
    title: "IT Team Lead",
    type: "Drop-By",
    notes: "Dropped by to follow up on outstanding support tickets. Discussed shared regional support setup now that the Alan B Miller facility is operational. Front desk was short-staffed celebrating Hospital Week (40th anniversary). Shared congratulations but IT team was tied up.",
    outcome: "Neutral",
    nextAction: "Follow up virtually next week re: cart maintenance agreements",
    beds: 159,
    zone: 2,
    source: "CRM Q1 Meetings Log"
  },
  {
    id: "act-8",
    date: "2026-03-17",
    account: "Jackson Health System",
    contact: "Daniel Rojas",
    title: "IT Director",
    type: "Scheduled Visit",
    notes: "Scheduled in-person meeting cancelled at the last minute because Daniel's daughter was sick. However, had Michael Baker help draft a gorgeous 3D rendering of a customized telehealth cart to align with their new tower goals. Sent rendering to keep momentum.",
    outcome: "Needs Follow-Up",
    nextAction: "Send telehealth rendering follow-up email; secure in-person reschedule",
    beds: 2400,
    zone: 4,
    source: "CRM March Log"
  },
  {
    id: "act-9",
    date: "2026-03-17",
    account: "Jackson Health System",
    contact: "Daniel Cruz",
    title: "Telehealth Engineer",
    type: "Scheduled Visit",
    notes: "Met with Daniel Cruz. He is highly motivated and impressed by our Samsung interactive display demo. Exploring buying multiple units to distribute across outpatient clinics.",
    outcome: "Positive",
    nextAction: "Submit formal Samsung AV proposal to Daniel Cruz with bulk discount options",
    beds: 2400,
    zone: 4,
    source: "CRM March Log"
  },
  {
    id: "act-10",
    date: "2026-03-17",
    account: "Jackson Health System",
    contact: "Raul Ordonez",
    title: "Vendor Relations / Compliance",
    type: "Drop-By",
    notes: "Discussed vendor consolidation plans with Raul. He wants to reduce active vendor headcount. Chris Colon (Axis security) is working on another project here, so we must tread carefully. Axis provided their existing product list to assist us.",
    outcome: "Neutral",
    nextAction: "Analyze Axis product list; map consolidation solutions for Raul",
    beds: 2400,
    zone: 4,
    source: "CRM March Log"
  },
  {
    id: "act-11",
    date: "2026-03-16",
    account: "Baptist Hospital of Miami",
    contact: "Connie Chan",
    title: "Director of Pharmacy",
    type: "Drop-By",
    notes: "Connected on LinkedIn regarding incoming pharmacy cart projects. IT department representatives were unavailable. Planning to drop by again when in Zone 4.",
    outcome: "Needs Follow-Up",
    nextAction: "In-person touch point scheduled next Monday; connect with primary Pharmacy IT Lead",
    beds: 728,
    zone: 4,
    source: "CRM March Log"
  },
  {
    id: "act-12",
    date: "2026-02-12",
    account: "Jupiter Medical Center",
    contact: "Kevin Olson",
    title: "Chief Information Officer",
    type: "Drop-By",
    notes: "Attempted to meet with Kevin. Met with Fredric Simpson (Facilities/Security). Unfortunately Fredric has a strict, long-term relationship with his physical security vendor (which he refused to name) and noted he does not handle IT. Emphasized we must reach Kevin Olson or his direct IT reports.",
    outcome: "Needs Follow-Up",
    nextAction: "Request warm referral through local networks to connect directly with Kevin Olson",
    beds: 163,
    zone: 1,
    source: "CRM Q1 Meetings Log"
  },
  {
    id: "act-13",
    date: "2026-02-12",
    account: "Palm Beach Gardens Medical Center",
    contact: "Sergio Valvez",
    title: "IT Manager (Contracted MSDL)",
    type: "Drop-By",
    notes: "Excellent conversation with Sergio. Gained valuable competitive intelligence: Sophos licenses are expiring. This is a massive opportunity to displace our competitor, Anthony Manso. Sergio is open to alternative proposals.",
    outcome: "Positive",
    nextAction: "Build a highly competitive security/network endpoint displacement package highlighting Sophos integration",
    beds: 204,
    zone: 2,
    source: "CRM Q1 Meetings Log"
  },
  {
    id: "act-14",
    date: "2026-02-12",
    account: "JFK Medical Center",
    contact: "Beverly Lindsay",
    title: "Director of Operating Room",
    type: "Drop-By",
    notes: "Met Beverly Lindsay in the surgical wing. Discussed barcode scanner hardware refresh cycles. They currently utilize a mix of Honeywell and Code Corp scanners. Proposed an on-site demo of the ruggedized Code barcode series.",
    outcome: "Positive",
    nextAction: "Coordinate on-site demo unit shipping; schedule demonstration calendar",
    beds: 424,
    zone: 2,
    source: "CRM Q1 Meetings Log"
  },
  {
    id: "act-15",
    date: "2026-02-10",
    account: "Wellington Regional Medical Center",
    contact: "Keith Pearson",
    title: "Facilities & Assets Manager",
    type: "Scheduled",
    notes: "Met Keith Pearson briefly to introduce myself and put a face to the name. Discussed the current age of their computer/nursing carts. Keith noted many units are failing, losing battery charge rapidly, or require immediate maintenance. Offered to perform a comprehensive cart inventory and wellness health check-up.",
    outcome: "Positive",
    nextAction: "Draft Cart Inventory Audit spreadsheet; secure dates for on-site tech walk-through",
    beds: 159,
    zone: 2,
    source: "CRM Q1 Meetings Log"
  }
];

const initialExpenses = [
  { period: "4/20/2026 to 4/26/2026", purpose: "Baptist Cart Week - Deployment audit and customer alignments in multiple sites", zones: "Zones 1, 2, 3", amount: 1070.31, status: "COMPLETED" },
  { period: "4/13/2026 to 4/19/2026", purpose: "On-site customer meetings, inventory checks & drop-bys on West Coast", zones: "Zones 5, 6", amount: 907.39, status: "COMPLETED" },
  { period: "3/30/2026 to 4/05/2026", purpose: "Hospital Week prep & regional IT leadership alignment dinners", zones: "Zones 2, 3", amount: 831.95, status: "COMPLETED" },
  { period: "3/23/2026 to 3/29/2026", purpose: "Regional drop-bys, contract expansions, and trial delivery audits", zones: "Zones 3, 4, 5", amount: 915.80, status: "COMPLETED" },
  { period: "3/16/2026 to 3/22/2026", purpose: "East Coast clinic drops and custom cart presentation travels", zones: "Zones 1, 3", amount: 616.42, status: "COMPLETED" },
  { period: "2/16/2026 to 2/22/2026", purpose: "Miami-Dade heavy territory drop-bys & Samsung interactive AV demos", zones: "Zone 4", amount: 1576.65, status: "COMPLETED" },
  { period: "2/09/2026 to 2/15/2026", purpose: "South Florida Medical AE regional drop-bys and scanner trials", zones: "Zones 1, 4", amount: 235.76, status: "COMPLETED" },
  { period: "1/26/2026 to 2/01/2026", purpose: "S.FL Med AE - Mid-territory meetings and hardware inventory drops", zones: "Zones 2, 3", amount: 721.76, status: "COMPLETED" },
  { period: "1/19/2026 to 1/25/2026", purpose: "Drop-bys and support ticket audits in Broward county hospitals", zones: "Zone 3", amount: 159.94, status: "COMPLETED" },
  { period: "1/12/2026 to 1/18/2026", purpose: "AE travels across Southern and Southwest coastlines", zones: "Zones 4, 5", amount: 728.00, status: "COMPLETED" },
  { period: "12/29/2025 to 1/04/2026", purpose: "Year-end hospital executive check-ins and holiday coverage", zones: "Zone 4", amount: 581.59, status: "REJECTED", reason: "Missing receipt logs for meals overages" },
  { period: "12/22/2025 to 12/28/2025", purpose: "Holiday emergency support travels and critical spares drops", zones: "Zone 1", amount: 134.40, status: "COMPLETED" },
  { period: "12/15/2025 to 12/21/2025", purpose: "Q4 wrap meetings, client onboarding, and thank you runs", zones: "Zone 3", amount: 547.68, status: "COMPLETED" }
];

const financialKpis = [
  { month: "January", projected: 20000, actual: 18500, diff: -1500, var: -7.50, opex: 15000, profit: 3500, margin: 18.92 },
  { month: "February", projected: 22000, actual: 23000, diff: 1000, var: 4.55, opex: 16000, profit: 7000, margin: 30.43 },
  { month: "March", projected: 25000, actual: 24500, diff: -500, var: -2.00, opex: 17000, profit: 7500, margin: 30.61 },
  { month: "April", projected: 30000, actual: 32000, diff: 2000, var: 6.67, opex: 18000, profit: 14000, margin: 43.75 },
  { month: "May", projected: 28000, actual: 27000, diff: -1000, var: -3.57, opex: 16500, profit: 10500, margin: 38.89 }
];

const ciscoAMs = [
  { account: "Jackson Health System", am: "Vince Merlo", email: "vmerlo@cisco.com" },
  { account: "Memorial Healthcare System", am: "Brian Avery", email: "bravery@cisco.com" },
  { account: "Baptist Health South Florida", am: "Mark Rittiner", email: "mrittine@cisco.com" },
  { account: "HCA East Florida Division", am: "Kimberly Royal", email: "kimbwhit@cisco.com" },
  { account: "Palm Beach Health Network (Tenet)", am: "Brian Avery", email: "bravery@cisco.com" },
  { account: "Orlando Health", am: "Matt Teague", email: "mateague@cisco.com" },
  { account: "Lee Health", am: "Richelle Selzer", email: "riselzer@cisco.com" },
  { account: "NCH Healthcare System", am: "Simon Wade", email: "simwade@cisco.com" },
  { account: "Cleveland Clinic Florida", am: "Brandon Mathie", email: "bmathie@cisco.com" },
  { account: "Mount Sinai Medical Center (Miami Beach)", am: "Anthony Forina", email: "aforina@cisco.com" }
];

const zoneDescriptions = {
  1: { name: "Zone 1: Treasure Coast / North", regions: "Martin, St. Lucie, Indian River Counties", color: "bg-sky-50 text-sky-800 border-sky-100" },
  2: { name: "Zone 2: Palm Beach County", regions: "West Palm Beach, Jupiter, Gardens, Atlantis", color: "bg-blue-50 text-blue-800 border-blue-100" },
  3: { name: "Zone 3: Broward County", regions: "Fort Lauderdale, Hollywood, Imperial Point", color: "bg-indigo-50 text-indigo-800 border-indigo-100" },
  4: { name: "Zone 4: Miami-Dade County", regions: "Miami, Coral Gables, Hialeah, South Miami", color: "bg-emerald-50 text-emerald-800 border-emerald-100" },
  5: { name: "Zone 5: Southwest Coast", regions: "Fort Myers, Naples, Lee County, Collier County", color: "bg-amber-50 text-amber-800 border-amber-100" },
  6: { name: "Zone 6: West Gulf Coast", regions: "Sarasota, Bradenton, St. Petersburg, Pinellas", color: "bg-rose-50 text-rose-800 border-rose-100" }
};

export default function App() {
  const [activeTab, setActiveTab] = useState('dashboard');
  const [accounts, setAccounts] = useState(initialAccounts);
  const [activities, setActivities] = useState(initialActivities);
  const [expenses, setExpenses] = useState(initialExpenses);
  const [user, setUser] = useState(null);
  
  // Search & Filter State
  const [searchTerm, setSearchTerm] = useState('');
  const [filterZone, setFilterZone] = useState('all');
  const [filterOutcome, setFilterOutcome] = useState('all');
  const [filterType, setFilterType] = useState('all');
  const [selectedActivity, setSelectedActivity] = useState(null);
  const [selectedZone, setSelectedZone] = useState(4); // default to Miami Zone 4
  
  // Add Log Modal/Form State
  const [showAddModal, setShowAddModal] = useState(false);
  const [newLog, setNewLog] = useState({
    account: '',
    contact: '',
    title: '',
    type: 'Meeting',
    notes: '',
    outcome: 'Positive',
    nextAction: '',
    beds: 100,
    zone: 4,
    source: 'Manual Rep Input'
  });

  // Cisco Planner State
  const [selectedCiscoAccount, setSelectedCiscoAccount] = useState('Jackson Health System');
  const [selectedCiscoTopic, setSelectedCiscoTopic] = useState('AI Infrastructure Refresh');
  const [generatedEmail, setGeneratedEmail] = useState('');
  const [copySuccess, setCopySuccess] = useState(false);

  // Cloud Sync Status Banner
  const [syncStatus, setSyncStatus] = useState('local'); // 'local', 'syncing', 'synced'
  const [syncMessage, setSyncMessage] = useState('Offline safe-mode enabled. Local cache active.');

  // Firebase Auth Setup (RULE 3)
  useEffect(() => {
    if (!auth) return;
    const initAuth = async () => {
      try {
        if (initialAuthToken) {
          await signInWithCustomToken(auth, initialAuthToken);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Firebase Authentication Error:", error);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (usr) => {
      if (usr) {
        setUser(usr);
        setSyncStatus('synced');
        setSyncMessage(`Secure cloud synchronization activated.`);
      }
    });
    return () => unsubscribe();
  }, []);

  // Firebase Firestore Setup (RULE 1, 2, 3)
  useEffect(() => {
    if (!db || !user) return;
    setSyncStatus('syncing');

    const publicDataCollection = collection(db, 'artifacts', appId, 'public', 'data', 'activities');
    const unsubscribe = onSnapshot(publicDataCollection, (snapshot) => {
      const remoteLogs = [];
      snapshot.forEach((doc) => {
        remoteLogs.push({ id: doc.id, ...doc.data() });
      });
      if (remoteLogs.length > 0) {
        setActivities((prev) => {
          const merged = [...prev];
          remoteLogs.forEach((remote) => {
            if (!merged.some((l) => l.id === remote.id)) {
              merged.unshift(remote);
            }
          });
          return merged;
        });
      }
      setSyncStatus('synced');
    }, (error) => {
      console.error("Firestore loading error:", error);
      setSyncStatus('local');
    });

    return () => unsubscribe();
  }, [user]);

  // Handle manual additions
  const handleAddLog = async (e) => {
    e.preventDefault();
    if (!newLog.account || !newLog.contact) {
      alert("Please fill out Account and Contact names.");
      return;
    }

    const logWithId = {
      ...newLog,
      id: `act-${Date.now()}`,
      date: new Date().toISOString().split('T')[0],
      beds: parseInt(newLog.beds) || 0,
      zone: parseInt(newLog.zone) || 4
    };

    setActivities([logWithId, ...activities]);
    setShowAddModal(false);

    if (db && user) {
      try {
        const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'activities', logWithId.id);
        await setDoc(docRef, logWithId);
        setSyncStatus('synced');
      } catch (err) {
        console.error("Cloud saving error:", err);
      }
    }

    setNewLog({
      account: '',
      contact: '',
      title: '',
      type: 'Meeting',
      notes: '',
      outcome: 'Positive',
      nextAction: '',
      beds: 100,
      zone: 4,
      source: 'Manual Rep Input'
    });
  };

  // Generate Partnership Email Template for Cisco AMs
  useEffect(() => {
    const amInfo = ciscoAMs.find(c => c.account === selectedCiscoAccount);
    if (!amInfo) return;

    let body = "";
    if (selectedCiscoTopic === 'AI Infrastructure Refresh') {
      body = `Hi ${amInfo.am || 'Partner'},\n\nI hope you're doing well. I am the Howard Technology Healthcare AE representing our South Florida portfolio. I am planning an upcoming meeting at ${selectedCiscoAccount} regarding expanding their clinical computing terminals and incorporating our joint AI-enabled infrastructure.\n\nI know you are the aligned Cisco Account Manager here. I want to sync with you on their active network capacity so we can align our server recommendations with Cisco security frameworks (including HHS cybersecurity compliance protocols).\n\nDo you have 10 minutes for a quick introductory alignment call this week?\n\nBest regards,\nAlexandra Gulledge\nSenior Medical Sales Representative, South Florida Territory`;
    } else if (selectedCiscoTopic === 'Point-of-Care Cart Upgrade') {
      body = `Hi ${amInfo.am || 'Partner'},\n\nI hope this email finds you well. I am working with the nursing directors and IT teams at ${selectedCiscoAccount} on an exciting cart health audit and equipment upgrade plan. Many of their existing carts suffer from legacy connectivity dropouts.\n\nWe want to bundle our next-generation point-of-care medical carts with Cisco secure wireless modules to ensure seamless roaming across their clinical floors. I would love to co-develop this presentation to the Chief Medical Officer.\n\nLet me know your availability for a joint pipeline review.\n\nWarmly,\nAlexandra Gulledge\nSenior Medical Sales Representative, South Florida Territory`;
    } else {
      body = `Hi ${amInfo.am || 'Partner'},\n\nI'm reaching out to align on our accounts at ${selectedCiscoAccount}. We have active engagements discussing cybersecurity, endpoint asset tracking, and hardware modernization. I want to review our local pipeline and explore where we can team up on co-selling opportunities to consolidate vendor footprint for the CIO.\n\nLet's jump on a quick call to map out mutual contacts.\n\nSincerely,\nAlexandra Gulledge\nSenior Medical Sales Representative, South Florida Territory`;
    }

    setGeneratedEmail(body);
  }, [selectedCiscoAccount, selectedCiscoTopic]);

  const copyToClipboard = () => {
    const el = document.createElement('textarea');
    el.value = generatedEmail;
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);
    setCopySuccess(true);
    setTimeout(() => setCopySuccess(false), 2500);
  };

  const triggerSyncSetup = async () => {
    setSyncStatus('syncing');
    setSyncMessage("Connecting to secure servers...");
    setTimeout(() => {
      setSyncStatus('synced');
      setSyncMessage("Synchronized with Howard Cloud network securely.");
    }, 1500);
  };

  const handleUpdateProductStatus = (accountId, category, currentStatus) => {
    const statusCycle = { 'none': 'pipeline', 'pipeline': 'active', 'active': 'none' };
    const nextStatus = statusCycle[currentStatus];

    setAccounts(prev => prev.map(acc => {
      if (acc.id === accountId) {
        return {
          ...acc,
          products: {
            ...acc.products,
            [category]: nextStatus
          }
        };
      }
      return acc;
    }));
  };

  const filteredActivities = activities.filter(act => {
    const matchesSearch = 
      act.account.toLowerCase().includes(searchTerm.toLowerCase()) ||
      act.contact.toLowerCase().includes(searchTerm.toLowerCase()) ||
      act.notes.toLowerCase().includes(searchTerm.toLowerCase()) ||
      act.title.toLowerCase().includes(searchTerm.toLowerCase());
    
    const matchesZone = filterZone === 'all' ? true : act.zone === parseInt(filterZone);
    const matchesOutcome = filterOutcome === 'all' ? true : act.outcome === filterOutcome;
    const matchesType = filterType === 'all' ? true : act.type === filterType;

    return matchesSearch && matchesZone && matchesOutcome && matchesType;
  });

  const activeContractsCount = accounts.reduce((acc, curr) => {
    const actives = Object.values(curr.products).filter(p => p === 'active').length;
    return acc + actives;
  }, 0);

  const pipelineCount = accounts.reduce((acc, curr) => {
    const pipelines = Object.values(curr.products).filter(p => p === 'pipeline').length;
    return acc + pipelines;
  }, 0);

  const totalBedsRepresented = accounts.reduce((acc, curr) => acc + curr.beds, 0);

  const totalExpensesCount = expenses
    .filter(e => e.status === 'COMPLETED')
    .reduce((acc, curr) => acc + curr.amount, 0);

  const averageWeeklyExpense = totalExpensesCount / expenses.filter(e => e.status === 'COMPLETED').length;

  return (
    <div className="min-h-screen bg-slate-50 text-slate-800 flex flex-col font-sans antialiased">
      
      {/* TOP BRANDING HEADER */}
      <header className="bg-white border-b border-slate-200/80 px-6 py-4 flex flex-col md:flex-row items-center justify-between gap-4 shrink-0 shadow-sm">
        <div className="flex items-center gap-3">
          <div className="bg-gradient-to-tr from-blue-600 to-sky-500 p-2.5 rounded-xl text-white shadow-md shadow-blue-200">
            <svg className="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2.5" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10" />
            </svg>
          </div>
          <div>
            <h1 className="text-xl font-extrabold tracking-tight text-slate-900 flex items-center gap-2">
              HOWARD <span className="text-blue-600 font-bold text-xs tracking-wider uppercase bg-blue-50/80 px-2.5 py-1 rounded border border-blue-100">HEALTHCARE SOLUTIONS</span>
            </h1>
            <p className="text-xs text-slate-500 font-medium">South Florida Territory Sales Hub</p>
          </div>
        </div>

        {/* REP CARD HEADER */}
        <div className="flex items-center gap-4 bg-slate-50 px-4 py-2 rounded-xl border border-slate-200/60 text-sm">
          <div className="w-8 h-8 rounded-full bg-blue-600 border-2 border-white flex items-center justify-center font-bold text-white shadow-sm">
            AG
          </div>
          <div className="text-left">
            <div className="font-bold text-slate-900 text-xs">Alexandra Gulledge</div>
            <div className="text-[10px] text-slate-400 tracking-wider uppercase font-semibold">Senior Medical Sales Executive</div>
          </div>
          <div className="h-6 w-[1px] bg-slate-200" />
          <div className="text-right">
            <div className="text-[10px] text-slate-400 uppercase font-bold">Region Assigned</div>
            <div className="text-blue-600 text-xs font-black">Zones 1–6 (South FL)</div>
          </div>
        </div>
      </header>

      {/* CLOUD CONNECTIVITY BAR */}
      <div className="bg-slate-100/60 px-6 py-2 border-b border-slate-200/80 flex items-center justify-between text-xs text-slate-500">
        <div className="flex items-center gap-2">
          <span className={`w-2 h-2 rounded-full ${syncStatus === 'synced' ? 'bg-emerald-500' : 'bg-amber-500 animate-pulse'}`} />
          <span>Sync Status: <strong className="text-slate-700">{syncStatus.toUpperCase()}</strong> — {syncMessage}</span>
        </div>
        {syncStatus === 'local' && (
          <button 
            onClick={triggerSyncSetup}
            className="text-blue-600 hover:text-blue-700 font-bold hover:underline flex items-center gap-1"
          >
            <svg className="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
            </svg>
            Connect Cloud Backup
          </button>
        )}
      </div>

      <div className="flex-1 flex flex-col md:flex-row overflow-hidden">
        
        {/* LIGHT SIDEBAR NAVIGATION */}
        <nav className="w-full md:w-64 bg-white border-r border-slate-200/80 shrink-0 p-4 flex flex-col gap-1.5 overflow-y-auto">
          <div className="text-slate-400 text-[10px] tracking-wider uppercase font-bold px-3 py-2">Operational Suite</div>
          
          <button 
            onClick={() => setActiveTab('dashboard')}
            className={`flex items-center gap-3 px-4 py-3 rounded-lg text-sm font-bold transition ${activeTab === 'dashboard' ? 'bg-blue-600 text-white shadow-sm shadow-blue-100' : 'text-slate-600 hover:bg-slate-50 hover:text-slate-900'}`}
          >
            <svg className="w-5 h-5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z" />
            </svg>
            Territory Overview
          </button>

          <button 
            onClick={() => setActiveTab('crm')}
            className={`flex items-center gap-3 px-4 py-3 rounded-lg text-sm font-bold transition ${activeTab === 'crm' ? 'bg-blue-600 text-white shadow-sm shadow-blue-100' : 'text-slate-600 hover:bg-slate-50 hover:text-slate-900'}`}
          >
            <svg className="w-5 h-5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z" />
            </svg>
            Meeting &amp; Activity CRM
            <span className={`ml-auto px-2 py-0.5 rounded-full text-[10px] font-extrabold ${activeTab === 'crm' ? 'bg-blue-700 text-white' : 'bg-slate-100 text-slate-600'}`}>
              {activities.length}
            </span>
          </button>

          <button 
            onClick={() => setActiveTab('map')}
            className={`flex items-center gap-3 px-4 py-3 rounded-lg text-sm font-bold transition ${activeTab === 'map' ? 'bg-blue-600 text-white shadow-sm shadow-blue-100' : 'text-slate-600 hover:bg-slate-50 hover:text-slate-900'}`}
          >
            <svg className="w-5 h-5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M9 20l-5.447-2.724A1 1 0 013 16.382V5.618a1 1 0 011.447-.894L9 7m0 13l6-3m-6 3V7m6 10l4.553 2.276A1 1 0 0021 18.382V7.618a1 1 0 00-.553-.894L15 4m0 13V4m0 0L9 7" />
            </svg>
            Territory &amp; Zones Map
          </button>

          <button 
            onClick={() => setActiveTab('matrix')}
            className={`flex items-center gap-3 px-4 py-3 rounded-lg text-sm font-bold transition ${activeTab === 'matrix' ? 'bg-blue-600 text-white shadow-sm shadow-blue-100' : 'text-slate-600 hover:bg-slate-50 hover:text-slate-900'}`}
          >
            <svg className="w-5 h-5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10" />
            </svg>
            White Space Opportunity
          </button>

          <button 
            onClick={() => setActiveTab('cisco')}
            className={`flex items-center gap-3 px-4 py-3 rounded-lg text-sm font-bold transition ${activeTab === 'cisco' ? 'bg-blue-600 text-white shadow-sm shadow-blue-100' : 'text-slate-600 hover:bg-slate-50 hover:text-slate-900'}`}
          >
            <svg className="w-5 h-5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M13 10V3L4 14h7v7l9-11h-7z" />
            </svg>
            Cisco Alliance Playbook
          </button>

          <button 
            onClick={() => setActiveTab('finance')}
            className={`flex items-center gap-3 px-4 py-3 rounded-lg text-sm font-bold transition ${activeTab === 'finance' ? 'bg-blue-600 text-white shadow-sm shadow-blue-100' : 'text-slate-600 hover:bg-slate-50 hover:text-slate-900'}`}
          >
            <svg className="w-5 h-5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2m0-8c1.11 0 2.08.402 2.599 1M12 8V7m0 1v8m0 0v1m0-1c-1.11 0-2.08-.402-2.599-1M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            Finance &amp; Expenses
          </button>

          {/* TERRITORY BRIEFING WIDGET */}
          <div className="mt-auto pt-6 border-t border-slate-100">
            <div className="bg-slate-50 p-4 rounded-xl border border-slate-200/50 text-xs">
              <div className="font-bold text-slate-700 mb-2 flex items-center gap-1.5">
                <span className="w-2.5 h-2.5 rounded-full bg-blue-500" />
                SFL Territory Vital Stats
              </div>
              <ul className="space-y-1.5 text-slate-500 text-[11px]">
                <li className="flex justify-between"><span>Target Sites:</span> <strong className="text-slate-700">{accounts.length}</strong></li>
                <li className="flex justify-between"><span>Combined Beds:</span> <strong className="text-slate-700">{totalBedsRepresented.toLocaleString()}</strong></li>
                <li className="flex justify-between"><span>Active Lines:</span> <strong className="text-emerald-600">{activeContractsCount}</strong></li>
                <li className="flex justify-between"><span>Open Proposals:</span> <strong className="text-amber-600">{pipelineCount}</strong></li>
              </ul>
            </div>
          </div>
        </nav>

        {/* MAIN BODY AREA */}
        <main className="flex-1 bg-slate-50 p-6 overflow-y-auto space-y-6 font-sans">
          
          {/* =======================================================
              1. TAB: DASHBOARD OVERVIEW
              ======================================================= */}
          {activeTab === 'dashboard' && (
            <div className="space-y-6 animate-fadeIn">
              
              {/* TARGET KPI METRICS ROW */}
              <div className="grid grid-cols-1 md:grid-cols-4 gap-5">
                <div className="bg-white p-5 rounded-xl border border-slate-200/80 shadow-sm hover:shadow transition relative overflow-hidden">
                  <div className="absolute top-0 right-0 p-4 opacity-5 text-blue-600">
                    <svg className="w-16 h-16" fill="currentColor" viewBox="0 0 24 24">
                      <path d="M19 12H5M19 12a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2" />
                    </svg>
                  </div>
                  <div className="text-[10px] text-slate-400 font-extrabold uppercase tracking-widest">Territory Bed Capacity</div>
                  <div className="text-3xl font-black text-slate-900 mt-1">{totalBedsRepresented.toLocaleString()}</div>
                  <div className="text-[10px] text-blue-600 font-semibold mt-1">Total active patient beds</div>
                </div>

                <div className="bg-white p-5 rounded-xl border border-slate-200/80 shadow-sm hover:shadow transition relative overflow-hidden">
                  <div className="text-[10px] text-slate-400 font-extrabold uppercase tracking-widest">Active Solutions Contracts</div>
                  <div className="text-3xl font-black text-emerald-600 mt-1">{activeContractsCount}</div>
                  <div className="text-[10px] text-emerald-600 font-semibold mt-1">Deployed contract units</div>
                </div>

                <div className="bg-white p-5 rounded-xl border border-slate-200/80 shadow-sm hover:shadow transition relative overflow-hidden">
                  <div className="text-[10px] text-slate-400 font-extrabold uppercase tracking-widest">Active Pipeline Funnel</div>
                  <div className="text-3xl font-black text-amber-500 mt-1">{pipelineCount}</div>
                  <div className="text-[10px] text-amber-600 font-semibold mt-1">Underway client pilots</div>
                </div>

                <div className="bg-white p-5 rounded-xl border border-slate-200/80 shadow-sm hover:shadow transition relative overflow-hidden">
                  <div className="text-[10px] text-slate-400 font-extrabold uppercase tracking-widest">Approved T&amp;E Expenses</div>
                  <div className="text-3xl font-black text-sky-600 mt-1">${totalExpensesCount.toLocaleString(undefined, {minimumFractionDigits: 2, maximumFractionDigits: 2})}</div>
                  <div className="text-[10px] text-slate-500 font-semibold mt-1">Weekly Avg: ${averageWeeklyExpense.toFixed(2)}</div>
                </div>
              </div>

              {/* ACTION ITEMS & TIMELINE CHART */}
              <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
                
                {/* URGENT COMPLIANCE & PILOT NOTIFICATIONS */}
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 lg:col-span-2 space-y-4 shadow-sm">
                  <div className="flex items-center justify-between border-b border-slate-100 pb-3">
                    <div className="flex items-center gap-2">
                      <span className="w-2.5 h-2.5 rounded-full bg-blue-600 animate-pulse" />
                      <h3 className="font-bold text-slate-900 text-sm">Priority Tasks &amp; Compliance Deadlines</h3>
                    </div>
                    <span className="text-[10px] bg-slate-50 text-slate-500 px-2 py-0.5 rounded border border-slate-200 font-semibold">SFL May–June 2026</span>
                  </div>

                  <div className="space-y-3 text-xs">
                    
                    {/* HHS Cybersecurity Compliance Goal */}
                    <div className="p-3 bg-red-50/50 border border-red-100 rounded-lg flex items-start gap-3">
                      <div className="bg-red-100 text-red-700 p-1.5 rounded shrink-0">
                        <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2.5" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
                        </svg>
                      </div>
                      <div className="flex-1">
                        <div className="flex items-center justify-between">
                          <span className="font-extrabold text-red-800">HHS Cybersecurity Performance Goal (CPG) #10 Audit</span>
                          <span className="text-[9px] bg-red-100/80 text-red-800 px-1.5 py-0.2 rounded font-mono font-bold">DUE JUNE 15, 2026</span>
                        </div>
                        <p className="text-slate-600 mt-1">Audit identified <strong>6 vendor partners</strong> lacking approved cybersecurity business agreements. Alexandra must deliver BAA compliance cure letters to procurement leads.</p>
                        <div className="mt-2 flex items-center gap-2">
                          <button onClick={() => { setActiveTab('crm'); setSearchTerm('HHS'); }} className="text-blue-600 hover:text-blue-700 hover:underline font-bold text-[11px]">Inspect Compliance Log →</button>
                        </div>
                      </div>
                    </div>

                    {/* Pending Trials */}
                    <div className="p-3 bg-blue-50/30 border border-blue-100 rounded-lg flex items-start gap-3">
                      <div className="bg-blue-50 text-blue-700 p-1.5 rounded shrink-0">
                        <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2m-6 9l2 2 4-4" />
                        </svg>
                      </div>
                      <div className="flex-1">
                        <div className="flex items-center justify-between">
                          <span className="font-extrabold text-slate-800">Memorial Regional Trial Proposal</span>
                          <span className="text-[9px] bg-blue-100/80 text-blue-800 px-1.5 py-0.2 rounded font-mono font-bold">URGENT</span>
                        </div>
                        <p className="text-slate-600 mt-1">Follow up on point-of-care mobile cart 3-month trial proposal. Sandra Torres (Director of Surgical Services) requested finalized volume discounts.</p>
                        <div className="mt-2">
                          <button onClick={() => { setActiveTab('crm'); setSearchTerm('Memorial'); }} className="text-blue-600 hover:text-blue-700 hover:underline font-bold text-[11px]">Review Meeting Details →</button>
                        </div>
                      </div>
                    </div>

                    {/* Front door bypass */}
                    <div className="p-3 bg-slate-50 border border-slate-200/60 rounded-lg flex items-start gap-3">
                      <div className="bg-slate-100 text-slate-600 p-1.5 rounded shrink-0">
                        <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M12 6V4m0 2a2 2 0 100 4m0-4a2 2 0 110 4m-6 8a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4m6 6v10m6-2a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4" />
                        </svg>
                      </div>
                      <div className="flex-1">
                        <span className="font-extrabold text-slate-800">Imperial Point Lobby Kiosk Opportunity</span>
                        <p className="text-slate-500 mt-1">Lobby congestion presents high chance to win Kiosk terminals. Lou advised to bypass the heavy front desk traffic directly through Chris Greco or IT facilities leads.</p>
                        <div className="mt-2">
                          <button onClick={() => { setActiveTab('crm'); setSearchTerm('Imperial'); }} className="text-blue-600 hover:text-blue-700 hover:underline font-bold text-[11px]">Read Strategy Notes</button>
                        </div>
                      </div>
                    </div>

                  </div>
                </div>

                {/* OUTCOMES CHANNELS CHART */}
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 flex flex-col justify-between shadow-sm">
                  <div>
                    <h3 className="font-bold text-slate-900 text-sm">Meeting Outcomes breakdown</h3>
                    <p className="text-slate-500 text-[11px]">SFL clinical account interaction results</p>
                  </div>
                  
                  {/* Modern Light Clean SVG Doughnut Chart */}
                  <div className="my-6 flex justify-center relative">
                    <svg className="w-36 h-36 transform -rotate-90" viewBox="0 0 36 36">
                      <circle cx="18" cy="18" r="15.915" fill="none" stroke="#f1f5f9" strokeWidth="3.5" />
                      {/* Positive slice (approx 55%) - royal blue */}
                      <circle cx="18" cy="18" r="15.915" fill="none" stroke="#2563eb" strokeWidth="4.2" strokeDasharray="55 100" strokeDashoffset="0" />
                      {/* Needs Follow-up (approx 25%) - light blue */}
                      <circle cx="18" cy="18" r="15.915" fill="none" stroke="#38bdf8" strokeWidth="4.2" strokeDasharray="25 100" strokeDashoffset="-55" />
                      {/* Neutral/Open (approx 20%) - slate gray */}
                      <circle cx="18" cy="18" r="15.915" fill="none" stroke="#cbd5e1" strokeWidth="4.2" strokeDasharray="20 100" strokeDashoffset="-80" />
                    </svg>
                    
                    <div className="absolute inset-0 flex flex-col items-center justify-center">
                      <span className="text-2xl font-black text-slate-950">{activities.length}</span>
                      <span className="text-[9px] text-slate-400 font-bold uppercase tracking-widest">Logs</span>
                    </div>
                  </div>

                  {/* Clean Light Legend */}
                  <div className="space-y-1.5 text-xs text-slate-500 border-t border-slate-50 pt-3">
                    <div className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <span className="w-2.5 h-2.5 rounded bg-blue-600" />
                        <span>Positive Progress</span>
                      </div>
                      <strong className="text-slate-800">55%</strong>
                    </div>
                    <div className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <span className="w-2.5 h-2.5 rounded bg-sky-400" />
                        <span>Needs Follow-Up</span>
                      </div>
                      <strong className="text-slate-800">25%</strong>
                    </div>
                    <div className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <span className="w-2.5 h-2.5 rounded bg-slate-300" />
                        <span>Neutral / Audits</span>
                      </div>
                      <strong className="text-slate-800">20%</strong>
                    </div>
                  </div>
                </div>

              </div>

              {/* REVENUE PERFORMANCE CHART */}
              <div className="bg-white border border-slate-200/80 rounded-xl p-5 shadow-sm">
                <div className="flex flex-col md:flex-row md:items-center justify-between gap-4 mb-4">
                  <div>
                    <h3 className="font-bold text-slate-900">2026 Monthly Performance Track</h3>
                    <p className="text-slate-500 text-xs">Comparing actual closed sales against targeted regional projections</p>
                  </div>
                  <div className="flex items-center gap-4 text-xs">
                    <div className="flex items-center gap-1.5">
                      <span className="w-3 h-3 bg-slate-200 rounded-sm" />
                      <span className="text-slate-500">Projected Goals</span>
                    </div>
                    <div className="flex items-center gap-1.5">
                      <span className="w-3 h-3 bg-blue-600 rounded-sm" />
                      <span className="text-slate-500">Actual Revenue</span>
                    </div>
                  </div>
                </div>

                {/* SVG Revenue Bar Chart (Clean Light Styling) */}
                <div className="h-44 flex items-end justify-between gap-3 border-b border-slate-100 pt-4 pb-1">
                  {financialKpis.map((k, i) => {
                    const maxVal = 35000;
                    const projHeight = (k.projected / maxVal) * 100;
                    const actHeight = (k.actual / maxVal) * 100;
                    const isPositive = k.diff >= 0;

                    return (
                      <div key={i} className="flex-1 flex flex-col items-center h-full group relative">
                        <div className="absolute -top-10 bg-slate-900 text-white rounded-lg px-2.5 py-1 text-[10px] hidden group-hover:block z-10 shadow-lg">
                          <div>Projected: ${k.projected.toLocaleString()}</div>
                          <div>Actual: <strong className="text-sky-300">${k.actual.toLocaleString()}</strong></div>
                          <div className={isPositive ? "text-emerald-400" : "text-rose-400"}>Var: {k.var}%</div>
                        </div>

                        <div className="flex items-end justify-center gap-1.5 w-full h-full">
                          <div className="w-1/4 bg-slate-100 rounded-t" style={{ height: `${projHeight}%` }} />
                          <div className="w-1/4 bg-blue-600 hover:bg-blue-500 rounded-t transition" style={{ height: `${actHeight}%` }} />
                        </div>

                        <span className="text-[11px] text-slate-500 font-bold mt-2">{k.month}</span>
                      </div>
                    );
                  })}
                </div>
              </div>

            </div>
          )}

          {/* =======================================================
              2. TAB: CRM SYSTEM MEETINGS
              ======================================================= */}
          {activeTab === 'crm' && (
            <div className="space-y-6 animate-fadeIn">
              
              {/* ADVANCED FILTERING & CONTROLS */}
              <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-4 shadow-sm">
                <div className="flex flex-col md:flex-row items-center justify-between gap-4">
                  <div>
                    <h2 className="text-lg font-extrabold text-slate-900">South Florida Client Interactions</h2>
                    <p className="text-xs text-slate-500 font-medium">Search, filter, and audit regional healthcare facility meetings and drop-by details</p>
                  </div>
                  
                  <div className="w-full md:w-auto">
                    <button 
                      onClick={() => setShowAddModal(true)}
                      className="bg-blue-600 hover:bg-blue-700 text-white font-extrabold py-2 px-4 rounded-lg text-xs flex items-center gap-1.5 shadow transition-all w-full justify-center"
                    >
                      <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2.5" d="M12 4v16m8-8H4" />
                      </svg>
                      Log CRM Encounter
                    </button>
                  </div>
                </div>

                {/* Filter Matrix */}
                <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 gap-3">
                  <div className="relative text-xs">
                    <span className="absolute inset-y-0 left-0 pl-3 flex items-center text-slate-400">
                      <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
                      </svg>
                    </span>
                    <input 
                      type="text" 
                      placeholder="Search accounts, notes, contacts..." 
                      value={searchTerm}
                      onChange={(e) => setSearchTerm(e.target.value)}
                      className="w-full bg-slate-50 border border-slate-200 rounded-lg pl-9 pr-4 py-2 text-xs focus:outline-none focus:border-blue-500 placeholder-slate-400"
                    />
                  </div>

                  <div>
                    <select 
                      value={filterZone} 
                      onChange={(e) => setFilterZone(e.target.value)}
                      className="w-full bg-slate-50 border border-slate-200 rounded-lg px-3 py-2 text-xs focus:outline-none focus:border-blue-500 text-slate-600 font-semibold"
                    >
                      <option value="all">All SFL Zones</option>
                      <option value="1">Zone 1 (Treasure Coast)</option>
                      <option value="2">Zone 2 (Palm Beach)</option>
                      <option value="3">Zone 3 (Broward)</option>
                      <option value="4">Zone 4 (Miami-Dade)</option>
                      <option value="5">Zone 5 (Southwest Coast)</option>
                      <option value="6">Zone 6 (Sarasota/St. Pete)</option>
                    </select>
                  </div>

                  <div>
                    <select 
                      value={filterOutcome} 
                      onChange={(e) => setFilterOutcome(e.target.value)}
                      className="w-full bg-slate-50 border border-slate-200 rounded-lg px-3 py-2 text-xs focus:outline-none focus:border-blue-500 text-slate-600 font-semibold"
                    >
                      <option value="all">All Outcomes</option>
                      <option value="Positive">Positive (Progressing)</option>
                      <option value="Needs Follow-Up">Needs Follow-Up</option>
                      <option value="Neutral">Neutral / Audits</option>
                      <option value="Open">Open Compliance</option>
                    </select>
                  </div>

                  <div>
                    <select 
                      value={filterType} 
                      onChange={(e) => setFilterType(e.target.value)}
                      className="w-full bg-slate-50 border border-slate-200 rounded-lg px-3 py-2 text-xs focus:outline-none focus:border-blue-500 text-slate-600 font-semibold"
                    >
                      <option value="all">All Meeting Types</option>
                      <option value="Meeting">Meeting (In Person)</option>
                      <option value="Demo">Product Demo</option>
                      <option value="Drop-By">Drop-By Visit</option>
                      <option value="Scheduled Visit">Scheduled Visit</option>
                      <option value="Presentation">Compliance Presentation</option>
                    </select>
                  </div>
                </div>
              </div>

              {/* DATA TABLE */}
              <div className="bg-white border border-slate-200/80 rounded-xl overflow-hidden shadow-sm">
                <div className="overflow-x-auto">
                  <table className="w-full text-left text-xs text-slate-700 border-collapse">
                    <thead className="bg-slate-50 text-slate-400 font-bold uppercase tracking-wider text-[10px] border-b border-slate-200">
                      <tr>
                        <th className="py-3.5 px-4">Date</th>
                        <th className="py-3.5 px-4">Facility / Account</th>
                        <th className="py-3.5 px-4">Key Contact</th>
                        <th className="py-3.5 px-4">Type</th>
                        <th className="py-3.5 px-4">Zone</th>
                        <th className="py-3.5 px-4">Outcome</th>
                        <th className="py-3.5 px-4">Next Goal</th>
                        <th className="py-3.5 px-4 text-center">Inspect</th>
                      </tr>
                    </thead>
                    <tbody className="divide-y divide-slate-100">
                      {filteredActivities.length === 0 ? (
                        <tr>
                          <td colSpan="8" className="py-12 text-center text-slate-400 font-semibold bg-white">
                            No active entries found matching current parameters.
                          </td>
                        </tr>
                      ) : (
                        filteredActivities.map((act) => {
                          let outcomeBadge = "bg-slate-100 text-slate-600 border-slate-200";
                          if (act.outcome === 'Positive') outcomeBadge = "bg-emerald-50 text-emerald-700 border-emerald-200";
                          if (act.outcome === 'Needs Follow-Up') outcomeBadge = "bg-amber-50 text-amber-700 border-amber-200";
                          if (act.outcome === 'Open') outcomeBadge = "bg-red-50 text-red-700 border-red-200";

                          return (
                            <tr 
                              key={act.id} 
                              className={`hover:bg-blue-50/20 transition cursor-pointer ${selectedActivity?.id === act.id ? 'bg-blue-50/40 font-medium' : ''}`}
                              onClick={() => setSelectedActivity(act)}
                            >
                              <td className="py-3.5 px-4 font-mono text-slate-500 font-semibold">{act.date}</td>
                              <td className="py-3.5 px-4">
                                <div className="font-extrabold text-slate-900">{act.account}</div>
                                <div className="text-[10px] text-slate-400">Capacity: {act.beds || 'N/A'} beds</div>
                              </td>
                              <td className="py-3.5 px-4">
                                <div className="font-semibold text-slate-800">{act.contact}</div>
                                <div className="text-[10px] text-slate-400">{act.title}</div>
                              </td>
                              <td className="py-3.5 px-4">
                                <span className="bg-blue-50 text-blue-700 px-2 py-0.5 rounded border border-blue-100 text-[9px] font-bold uppercase">{act.type}</span>
                              </td>
                              <td className="py-3.5 px-4 font-bold text-slate-500">Zone {act.zone || '3'}</td>
                              <td className="py-3.5 px-4">
                                <span className={`px-2 py-0.5 rounded border text-[9px] font-bold uppercase ${outcomeBadge}`}>
                                  {act.outcome}
                                </span>
                              </td>
                              <td className="py-3.5 px-4 max-w-xs truncate text-slate-500 font-medium">{act.nextAction}</td>
                              <td className="py-3.5 px-4 text-center">
                                <button className="text-blue-600 hover:text-blue-700 font-bold bg-slate-50 p-1 rounded border border-slate-200/80">
                                  <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
                                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z" />
                                  </svg>
                                </button>
                              </td>
                            </tr>
                          );
                        })
                      )}
                    </tbody>
                  </table>
                </div>
              </div>

              {/* CRM DETAILS PANEL */}
              {selectedActivity && (
                <div className="bg-white border border-slate-200/80 rounded-xl p-6 space-y-4 shadow-md animate-fadeIn">
                  <div className="flex items-center justify-between border-b border-slate-100 pb-3">
                    <div>
                      <span className="text-[9px] font-extrabold uppercase tracking-wider text-blue-600 font-mono">Detailed Representative Insights</span>
                      <h3 className="text-base font-black text-slate-900">{selectedActivity.account}</h3>
                    </div>
                    <button 
                      onClick={() => setSelectedActivity(null)}
                      className="text-slate-400 hover:text-slate-600 text-xs font-bold"
                    >
                      Close Inspector
                    </button>
                  </div>

                  <div className="grid grid-cols-1 md:grid-cols-3 gap-6 text-xs leading-relaxed">
                    <div className="space-y-3">
                      <div>
                        <div className="text-slate-400 font-bold mb-0.5 uppercase text-[9px]">Decision Maker</div>
                        <div className="font-extrabold text-slate-900 text-sm">{selectedActivity.contact}</div>
                        <div className="text-blue-600 font-bold">{selectedActivity.title}</div>
                      </div>
                      <div className="pt-2">
                        <div className="text-slate-400 font-bold mb-0.5 uppercase text-[9px]">Encounter Specifics</div>
                        <div className="text-slate-600 space-y-1">
                          <div>Meeting Type: <strong className="text-slate-800">{selectedActivity.type}</strong></div>
                          <div>Territory Zone: <strong className="text-slate-800">Zone {selectedActivity.zone}</strong></div>
                          <div>Hospital Size: <strong className="text-slate-800">{selectedActivity.beds || 'N/A'} beds</strong></div>
                        </div>
                      </div>
                    </div>

                    <div className="md:col-span-2 space-y-4">
                      <div>
                        <div className="text-slate-400 font-bold mb-1 uppercase text-[9px]">On-site Observations &amp; Threat Intelligence</div>
                        <p className="bg-slate-50 p-4 rounded-lg text-slate-700 border border-slate-200/60 italic leading-relaxed">
                          "{selectedActivity.notes}"
                        </p>
                      </div>

                      <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                        <div className="p-3 bg-blue-50/50 border border-blue-100 rounded-lg">
                          <div className="text-blue-700 font-bold mb-1 uppercase text-[9px]">Assigned Next Action Item</div>
                          <div className="text-slate-700 font-bold">{selectedActivity.nextAction}</div>
                        </div>

                        <div className="p-3 bg-slate-50 border border-slate-200/60 rounded-lg">
                          <div className="text-slate-400 font-bold mb-1 uppercase text-[9px]">Source Log Document</div>
                          <div className="text-slate-600 font-bold">{selectedActivity.source || 'CRM Timeline'}</div>
                          <div className="text-[10px] text-slate-400 font-mono mt-1">Synced: {selectedActivity.date}</div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              )}

            </div>
          )}

          {/* =======================================================
              3. TAB: TERRITORY ZONE GEOGRAPHY
              ======================================================= */}
          {activeTab === 'map' && (
            <div className="space-y-6 animate-fadeIn">
              
              <div className="bg-white border border-slate-200/80 rounded-xl p-5 shadow-sm">
                <h2 className="text-lg font-black text-slate-900">Geographical Sales Territory Structure</h2>
                <p className="text-xs text-slate-500 font-medium">Explore clinical facilities, bed counts, and direct points of contact segmented by Alexandra's six zones</p>
              </div>

              <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
                
                {/* ZONE MATRIX PANEL */}
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-4 shadow-sm">
                  <h3 className="font-extrabold text-slate-900 text-sm">Select Regional Coverage</h3>
                  <p className="text-slate-500 text-xs">Filter by physical Florida coordinates to access regional healthcare listings:</p>

                  <div className="space-y-2">
                    {[1, 2, 3, 4, 5, 6].map((z) => (
                      <button
                        key={z}
                        onClick={() => setSelectedZone(z)}
                        className={`w-full p-3 rounded-lg border text-left transition flex items-center justify-between ${selectedZone === z ? 'bg-blue-600 text-white border-blue-400 shadow' : 'bg-slate-50 text-slate-700 border-slate-200/80 hover:bg-slate-100'}`}
                      >
                        <div>
                          <div className="font-extrabold text-xs">{zoneDescriptions[z].name}</div>
                          <div className={`text-[10px] ${selectedZone === z ? 'text-blue-100' : 'text-slate-400'} font-semibold mt-0.5`}>
                            {zoneDescriptions[z].regions}
                          </div>
                        </div>
                        <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2.5" d="M9 5l7 7-7 7" />
                        </svg>
                      </button>
                    ))}
                  </div>
                </div>

                {/* ACTIVE ZONE DETAILS */}
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 lg:col-span-2 space-y-4 shadow-sm">
                  <div className="flex items-center justify-between border-b border-slate-100 pb-3">
                    <div>
                      <span className="text-[9px] text-blue-600 font-mono font-extrabold uppercase">SFL Regional Operations</span>
                      <h3 className="text-base font-black text-slate-900">{zoneDescriptions[selectedZone].name}</h3>
                    </div>
                    <span className="text-xs font-bold text-slate-400 bg-slate-50 border border-slate-200 px-2 py-0.5 rounded">
                      Local Target Accounts: {accounts.filter(a => a.zone === selectedZone).length}
                    </span>
                  </div>

                  {/* Account display cards */}
                  <div className="space-y-3">
                    {accounts.filter(a => a.zone === selectedZone).length === 0 ? (
                      <div className="py-12 text-center text-slate-400 text-xs font-semibold bg-slate-50 border border-slate-100 rounded-xl">
                        No facilities listed inside this territorial zone. Expand scope or log client location.
                      </div>
                    ) : (
                      accounts.filter(a => a.zone === selectedZone).map((acc) => (
                        <div key={acc.id} className="p-4 bg-slate-50/50 border border-slate-200/60 rounded-xl space-y-3 hover:border-blue-300 transition-all">
                          <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-2">
                            <div>
                              <div className="flex items-center gap-2">
                                <span className="font-extrabold text-slate-950 text-sm">{acc.name}</span>
                                <span className="bg-blue-50 text-blue-700 px-1.5 py-0.2 rounded text-[9px] font-extrabold border border-blue-100 uppercase">{acc.type}</span>
                              </div>
                              <span className="text-[10px] text-slate-400 font-semibold font-mono">{acc.address}</span>
                            </div>
                            <div className="text-right">
                              <span className="text-xs font-extrabold text-blue-700 bg-blue-50 border border-blue-100 px-2.5 py-1 rounded-lg">{acc.beds} Beds</span>
                            </div>
                          </div>

                          <div className="bg-white p-3 rounded-lg border border-slate-200/50 text-xs">
                            <span className="text-[9px] text-slate-400 font-extrabold uppercase tracking-wider block mb-1.5">Aligned Contacts &amp; Buying Habits</span>
                            <div className="grid grid-cols-1 sm:grid-cols-2 gap-2">
                              {acc.pocs.map((poc, idx) => (
                                <div key={idx} className="bg-slate-50/50 p-2.5 rounded border border-slate-200/60 text-slate-600">
                                  <div className="font-extrabold text-slate-900 text-[11px]">{poc.name}</div>
                                  <div className="text-[9px] text-slate-400 font-semibold">{poc.title}</div>
                                  <div className="text-[9px] text-blue-600 font-bold italic mt-0.5">{poc.note}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                        </div>
                      ))
                    )}
                  </div>
                </div>

              </div>

            </div>
          )}

          {/* =======================================================
              4. TAB: PRODUCT WHITE SPACE GRID
              ======================================================= */}
          {activeTab === 'matrix' && (
            <div className="space-y-6 animate-fadeIn">
              
              <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-2 shadow-sm">
                <h2 className="text-lg font-black text-slate-900">Howard Product Coverage White Space Matrix</h2>
                <p className="text-xs text-slate-500 font-medium">Coordinate cross-selling tracks and identify portfolio gaps. Click any cell to cycle through status values:</p>
                
                <div className="flex flex-wrap gap-4 text-xs pt-2">
                  <div className="flex items-center gap-1.5">
                    <span className="w-3.5 h-3.5 rounded bg-emerald-500 shadow-sm" />
                    <span className="text-slate-600 font-medium">Active Sales Contract</span>
                  </div>
                  <div className="flex items-center gap-1.5">
                    <span className="w-3.5 h-3.5 rounded bg-amber-400 animate-pulse shadow-sm" />
                    <span className="text-slate-600 font-medium">Open Pilot / Proposal Discussion</span>
                  </div>
                  <div className="flex items-center gap-1.5">
                    <span className="w-3.5 h-3.5 rounded bg-slate-100 border border-slate-200/80" />
                    <span className="text-slate-400 font-medium">Untapped Market / No Contact</span>
                  </div>
                </div>
              </div>

              {/* GRID TABLE */}
              <div className="bg-white border border-slate-200/80 rounded-xl overflow-hidden shadow-sm">
                <div className="overflow-x-auto">
                  <table className="w-full text-left text-xs text-slate-700 border-collapse">
                    <thead className="bg-slate-50 text-slate-400 font-bold uppercase tracking-wider text-[10px] border-b border-slate-200">
                      <tr>
                        <th className="py-3 px-4 min-w-[210px]">Medical Facility</th>
                        <th className="py-3 px-2 text-center">Computing</th>
                        <th className="py-3 px-2 text-center">A/V Displays</th>
                        <th className="py-3 px-2 text-center">Network</th>
                        <th className="py-3 px-2 text-center">CCTV Sec</th>
                        <th className="py-3 px-2 text-center">Kiosks</th>
                        <th className="py-3 px-2 text-center">Software</th>
                        <th className="py-3 px-2 text-center">Clinical Carts</th>
                        <th className="py-3 px-2 text-center">Telehealth</th>
                        <th className="py-3 px-2 text-center">Patient Exp</th>
                        <th className="py-3 px-2 text-center">Infection</th>
                      </tr>
                    </thead>
                    <tbody className="divide-y divide-slate-100">
                      {accounts.map((acc) => (
                        <tr key={acc.id} className="hover:bg-slate-50 transition">
                          <td className="py-3 px-4 border-r border-slate-100">
                            <div className="font-extrabold text-slate-900">{acc.name}</div>
                            <div className="text-[9px] text-slate-400">Zone {acc.zone} • {acc.beds} beds</div>
                          </td>

                          {/* Computing */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'computing', acc.products.computing)}>
                              {acc.products.computing === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.computing === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.computing === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Audiovisual */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'audiovisual', acc.products.audiovisual)}>
                              {acc.products.audiovisual === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.audiovisual === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.audiovisual === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Network */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'network', acc.products.network)}>
                              {acc.products.network === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.network === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.network === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Security */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'security', acc.products.security)}>
                              {acc.products.security === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.security === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.security === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Kiosks */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'kiosks', acc.products.kiosks)}>
                              {acc.products.kiosks === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.kiosks === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.kiosks === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Software */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'software', acc.products.software)}>
                              {acc.products.software === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.software === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.software === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Mobile Carts */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'poc_carts', acc.products.poc_carts)}>
                              {acc.products.poc_carts === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.poc_carts === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.poc_carts === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Telehealth */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'telehealth', acc.products.telehealth)}>
                              {acc.products.telehealth === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.telehealth === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.telehealth === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Patient Experience */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'patient_exp', acc.products.patient_exp)}>
                              {acc.products.patient_exp === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.patient_exp === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.patient_exp === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                          {/* Infection Control */}
                          <td className="py-3 px-2 text-center">
                            <button onClick={() => handleUpdateProductStatus(acc.id, 'infection_control', acc.products.infection_control)}>
                              {acc.products.infection_control === 'active' && <span className="inline-block w-4.5 h-4.5 rounded bg-emerald-500 shadow-sm" />}
                              {acc.products.infection_control === 'pipeline' && <span className="inline-block w-4.5 h-4.5 rounded bg-amber-400 animate-pulse shadow-sm" />}
                              {acc.products.infection_control === 'none' && <span className="inline-block w-3.5 h-3.5 rounded bg-slate-100 hover:bg-slate-200 border border-slate-200" />}
                            </button>
                          </td>

                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </div>

            </div>
          )}

          {/* =======================================================
              5. TAB: CISCO ALLIANCE ALLY DRAFTING
              ======================================================= */}
          {activeTab === 'cisco' && (
            <div className="space-y-6 animate-fadeIn">
              
              <div className="bg-white border border-slate-200/80 rounded-xl p-5 shadow-sm">
                <h2 className="text-lg font-black text-slate-900">Howard &amp; Cisco Joint Field Alignment</h2>
                <p className="text-xs text-slate-500 font-medium">Coordinate physical secure networks and infrastructure proposals with local Cisco Account Managers (AMs)</p>
              </div>

              {/* AUDIENCE PLAYBOOK MESSAGE DECKS */}
              <div className="grid grid-cols-1 md:grid-cols-3 gap-5">
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-2 shadow-sm">
                  <div className="text-blue-600 text-xs font-extrabold uppercase tracking-wider">IT &amp; CIO Track</div>
                  <h4 className="font-bold text-slate-950">"Secure and Standardize Backend AI Platforms"</h4>
                  <p className="text-xs text-slate-500 leading-relaxed">
                    Prioritize data safety, active BAA compliance guarantees, Epic/Cerner wireless continuity, and endpoint security protection.
                  </p>
                </div>

                <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-2 shadow-sm">
                  <div className="text-sky-600 text-xs font-extrabold uppercase tracking-wider">Clinical Leaders Track</div>
                  <h4 className="font-bold text-slate-950">"Clinical Workflow Mobility &amp; Ergonomics"</h4>
                  <p className="text-xs text-slate-500 leading-relaxed">
                    Demonstrate direct impact on nursing staff, reliable barcode scanning, lightweight cart travel, and extended hot-swap battery lifecycles.
                  </p>
                </div>

                <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-2 shadow-sm">
                  <div className="text-slate-500 text-xs font-extrabold uppercase tracking-wider">C-Suite &amp; CFO Track</div>
                  <h4 className="font-bold text-slate-950">"Consolidated Healthcare vendor framework"</h4>
                  <p className="text-xs text-slate-500 leading-relaxed">
                    Emphasize procurement scaling, reduction of active third-party support agreements, and depreciable hardware investment values.
                  </p>
                </div>
              </div>

              {/* EMAIL TEMPLATING ENGINE */}
              <div className="bg-white border border-slate-200/80 rounded-xl p-6 grid grid-cols-1 lg:grid-cols-3 gap-6 shadow-sm">
                
                {/* SELECTOR FORM */}
                <div className="space-y-4">
                  <h3 className="font-bold text-slate-900 text-sm">Cisco AM Liaison Correspondence</h3>
                  <p className="text-slate-500 text-xs font-medium">Select a shared target and subject to draft a high-impact co-selling outreach request:</p>

                  <div className="space-y-3.5 text-xs">
                    <div>
                      <label className="block text-slate-500 font-bold mb-1">Target Sales Account</label>
                      <select 
                        value={selectedCiscoAccount} 
                        onChange={(e) => setSelectedCiscoAccount(e.target.value)}
                        className="w-full bg-slate-50 border border-slate-200 rounded-lg px-3 py-2 focus:outline-none focus:border-blue-500 text-slate-700 font-semibold"
                      >
                        {ciscoAMs.map((c, idx) => (
                          <option key={idx} value={c.account}>{c.account}</option>
                        ))}
                      </select>
                    </div>

                    <div>
                      <label className="block text-slate-500 font-bold mb-1">Discussion Topic / Goal</label>
                      <select 
                        value={selectedCiscoTopic} 
                        onChange={(e) => setSelectedCiscoTopic(e.target.value)}
                        className="w-full bg-slate-50 border border-slate-200 rounded-lg px-3 py-2 focus:outline-none focus:border-blue-500 text-slate-700 font-semibold"
                      >
                        <option value="AI Infrastructure Refresh">AI Infrastructure Refresh</option>
                        <option value="Point-of-Care Cart Upgrade">Point-of-Care Cart Upgrade</option>
                        <option value="General Account Alignment">General Account Alignment</option>
                      </select>
                    </div>

                    <div className="p-3.5 bg-blue-50/50 border border-blue-100 rounded-lg text-slate-600 space-y-1">
                      <div className="font-extrabold text-[9px] text-blue-800 uppercase tracking-wider">Assigned Cisco Partner</div>
                      <div className="text-slate-800 font-extrabold text-sm">{ciscoAMs.find(c => c.account === selectedCiscoAccount)?.am}</div>
                      <div className="text-xs font-mono text-blue-600 font-bold">{ciscoAMs.find(c => c.account === selectedCiscoAccount)?.email}</div>
                    </div>
                  </div>
                </div>

                {/* COPY GENERATOR */}
                <div className="lg:col-span-2 flex flex-col justify-between space-y-3">
                  <div className="flex-1">
                    <label className="block text-slate-400 text-xs font-extrabold mb-1">Outreach Draft (Modifiable)</label>
                    <textarea 
                      value={generatedEmail}
                      onChange={(e) => setGeneratedEmail(e.target.value)}
                      className="w-full h-64 bg-slate-50 border border-slate-200 rounded-lg p-4 text-xs focus:outline-none focus:border-blue-500 text-slate-700 font-mono leading-relaxed shadow-inner"
                    />
                  </div>

                  <div className="flex items-center justify-between border-t border-slate-50 pt-3">
                    <span className="text-[11px] text-slate-400 italic">This copy aligns with active field playbooks.</span>
                    <button 
                      onClick={copyToClipboard}
                      className="bg-blue-600 hover:bg-blue-700 text-white font-extrabold py-2 px-6 rounded-lg text-xs transition flex items-center gap-1.5 shadow-sm shadow-blue-200"
                    >
                      {copySuccess ? 'Copied to Clipboard!' : 'Copy to Clipboard'}
                    </button>
                  </div>
                </div>

              </div>

            </div>
          )}

          {/* =======================================================
              6. TAB: FINANCE & TRAVEL EXPENSE PROFILES
              ======================================================= */}
          {activeTab === 'finance' && (
            <div className="space-y-6 animate-fadeIn">
              
              <div className="bg-white border border-slate-200/80 rounded-xl p-5 shadow-sm">
                <h2 className="text-lg font-black text-slate-900">Regional Travel Expenses &amp; KPI Audits</h2>
                <p className="text-xs text-slate-500 font-medium">Cross-reference and monitor weekly on-site claim approvals, audit flags, and territory budgets</p>
              </div>

              {/* STATS PANELS */}
              <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                
                {/* CLAIM SUMMARY CARD */}
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 space-y-4 shadow-sm">
                  <h3 className="font-extrabold text-slate-900 text-sm border-b border-slate-50 pb-2">Approved Expense Claims</h3>
                  
                  <div className="space-y-3.5 text-xs text-slate-600">
                    <div className="flex justify-between border-b border-slate-100 pb-1.5">
                      <span>Submitted Log Weeks:</span>
                      <strong className="text-slate-800">{expenses.length} Weeks</strong>
                    </div>
                    <div className="flex justify-between border-b border-slate-100 pb-1.5">
                      <span>Claims Paid:</span>
                      <strong className="text-emerald-600 font-bold">
                        {expenses.filter(e => e.status === 'COMPLETED').length} Approved
                      </strong>
                    </div>
                    <div className="flex justify-between border-b border-slate-100 pb-1.5">
                      <span>Rejected Claims:</span>
                      <strong className="text-red-600 font-bold">
                        {expenses.filter(e => e.status === 'REJECTED').length} Flagged
                      </strong>
                    </div>
                    <div className="flex justify-between">
                      <span>Weekly Travel Expense Mean:</span>
                      <strong className="text-slate-900 font-black">${averageWeeklyExpense.toFixed(2)}</strong>
                    </div>
                  </div>
                </div>

                {/* ZONE TRAVEL DISTRIBUTION SUMMARY */}
                <div className="bg-white border border-slate-200/80 rounded-xl p-5 lg:col-span-2 space-y-4 shadow-sm">
                  <h3 className="font-extrabold text-slate-900 text-sm">Estimated Cumulative Travel Outlays by Territory Zone</h3>
                  <p className="text-slate-500 text-xs font-medium">Quantifying SFL healthcare travel spending vs. geography:</p>

                  <div className="grid grid-cols-2 sm:grid-cols-3 gap-3">
                    <div className="bg-slate-50 p-3 rounded-lg border border-slate-200/60">
                      <span className="text-[9px] text-slate-400 font-extrabold block uppercase">Zone 1 &amp; 2 (Palm Beach Network)</span>
                      <span className="text-sm font-black text-slate-900 mt-0.5 block">$1,850.50</span>
                    </div>
                    <div className="bg-slate-50 p-3 rounded-lg border border-slate-200/60">
                      <span className="text-[9px] text-slate-400 font-extrabold block uppercase">Zone 3 (Broward County System)</span>
                      <span className="text-sm font-black text-slate-900 mt-0.5 block">$2,450.80</span>
                    </div>
                    <div className="bg-slate-50 p-3 rounded-lg border border-slate-200/60">
                      <span className="text-[9px] text-slate-400 font-extrabold block uppercase">Zone 4 (Miami-Dade County)</span>
                      <span className="text-sm font-black text-slate-900 mt-0.5 block">$3,250.20</span>
                    </div>
                    <div className="bg-slate-50 p-3 rounded-lg border border-slate-200/60">
                      <span className="text-[9px] text-slate-400 font-extrabold block uppercase">Zone 5 &amp; 6 (West Florida Coast)</span>
                      <span className="text-sm font-black text-slate-900 mt-0.5 block">$1,454.56</span>
                    </div>
                  </div>
                </div>

              </div>

              {/* DETAILED LEDGER */}
              <div className="bg-white border border-slate-200/80 rounded-xl overflow-hidden shadow-sm">
                <div className="p-4 bg-slate-50 border-b border-slate-200 flex items-center justify-between">
                  <span className="font-extrabold text-slate-800 text-xs">Expense Claims History Log</span>
                  <div className="text-[11px] font-bold text-slate-500">
                    SFL Executive Target Metrics: <span className="text-blue-600">Actual Revenue: $378K</span> | <span className="text-indigo-600">Net Profit: $159.5K</span>
                  </div>
                </div>
                
                <div className="overflow-x-auto">
                  <table className="w-full text-left text-xs text-slate-700 border-collapse">
                    <thead className="bg-slate-50 text-slate-400 font-bold uppercase tracking-wider text-[10px] border-b border-slate-200">
                      <tr>
                        <th className="py-3 px-4">Period</th>
                        <th className="py-3 px-4">Focus &amp; Justification</th>
                        <th className="py-3 px-4">Zones Traveled</th>
                        <th className="py-3 px-4">Amount Claimed</th>
                        <th className="py-3 px-4">Status</th>
                      </tr>
                    </thead>
                    <tbody className="divide-y divide-slate-100">
                      {expenses.map((exp, idx) => (
                        <tr key={idx} className="hover:bg-slate-50 transition">
                          <td className="py-3.5 px-4 font-mono font-bold text-slate-500">{exp.period}</td>
                          <td className="py-3.5 px-4">
                            <div className="font-bold text-slate-800">{exp.purpose}</div>
                            {exp.reason && (
                              <div className="text-[10px] text-red-600 font-semibold bg-red-50/50 border border-red-100 p-1.5 rounded mt-1.5 max-w-md">
                                Audit Flag: {exp.reason}
                              </div>
                            )}
                          </td>
                          <td className="py-3.5 px-4 text-slate-500 font-bold">{exp.zones}</td>
                          <td className="py-3.5 px-4 font-black text-slate-900">${exp.amount.toFixed(2)}</td>
                          <td className="py-3.5 px-4">
                            <span className={`px-2 py-0.5 rounded border text-[9px] font-black uppercase ${exp.status === 'COMPLETED' ? 'bg-emerald-50 text-emerald-700 border-emerald-200' : 'bg-red-50 text-red-700 border-red-200'}`}>
                              {exp.status}
                            </span>
                          </td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </div>

            </div>
          )}

        </main>
      </div>

      {/* =======================================================
          MODAL: ADD NEW LOG
          ======================================================= */}
      {showAddModal && (
        <div className="fixed inset-0 z-50 bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4">
          <div className="bg-white border border-slate-200 rounded-xl p-6 max-w-lg w-full space-y-4 shadow-2xl animate-scaleUp text-slate-700">
            
            <div className="flex items-center justify-between border-b border-slate-100 pb-3">
              <h3 className="font-extrabold text-slate-900 text-base">Log Custom SFL CRM Entry</h3>
              <button 
                onClick={() => setShowAddModal(false)}
                className="text-slate-400 hover:text-slate-600 text-xs font-bold"
              >
                Cancel
              </button>
            </div>

            <form onSubmit={handleAddLog} className="space-y-4 text-xs">
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-slate-500 font-bold mb-1">Facility Name *</label>
                  <input 
                    type="text" 
                    required
                    placeholder="e.g. Jackson Health System"
                    value={newLog.account}
                    onChange={(e) => setNewLog({...newLog, account: e.target.value})}
                    className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-800 font-semibold"
                  />
                </div>

                <div>
                  <label className="block text-slate-500 font-bold mb-1">Key Contact *</label>
                  <input 
                    type="text" 
                    required
                    placeholder="e.g. Sandra Torres"
                    value={newLog.contact}
                    onChange={(e) => setNewLog({...newLog, contact: e.target.value})}
                    className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-800 font-semibold"
                  />
                </div>
              </div>

              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-slate-500 font-bold mb-1">Contact Job Title</label>
                  <input 
                    type="text" 
                    placeholder="e.g. Director of Operating Room"
                    value={newLog.title}
                    onChange={(e) => setNewLog({...newLog, title: e.target.value})}
                    className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-800 font-semibold"
                  />
                </div>

                <div>
                  <label className="block text-slate-500 font-bold mb-1">Territory Zone</label>
                  <select 
                    value={newLog.zone}
                    onChange={(e) => setNewLog({...newLog, zone: parseInt(e.target.value)})}
                    className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-700 font-semibold"
                  >
                    <option value={1}>Zone 1 (Treasure Coast)</option>
                    <option value={2}>Zone 2 (Palm Beach)</option>
                    <option value={3}>Zone 3 (Broward)</option>
                    <option value={4}>Zone 4 (Miami-Dade)</option>
                    <option value={5}>Zone 5 (Southwest Coast)</option>
                    <option value={6}>Zone 6 (Sarasota/St. Pete)</option>
                  </select>
                </div>
              </div>

              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-slate-500 font-bold mb-1">Meeting Category</label>
                  <select 
                    value={newLog.type}
                    onChange={(e) => setNewLog({...newLog, type: e.target.value})}
                    className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-700 font-semibold"
                  >
                    <option value="Meeting">Meeting (In Person)</option>
                    <option value="Demo">Product Demo</option>
                    <option value="Drop-By">Drop-By Encounter</option>
                    <option value="Scheduled Visit">Scheduled Visit</option>
                  </select>
                </div>

                <div>
                  <label className="block text-slate-500 font-bold mb-1">Encounter Outcome</label>
                  <select 
                    value={newLog.outcome}
                    onChange={(e) => setNewLog({...newLog, outcome: e.target.value})}
                    className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-700 font-semibold"
                  >
                    <option value="Positive">Positive Progress</option>
                    <option value="Needs Follow-Up">Needs Follow-Up</option>
                    <option value="Neutral">Neutral / Audits</option>
                  </select>
                </div>
              </div>

              <div>
                <label className="block text-slate-500 font-bold mb-1">Action Goal Deadline</label>
                <input 
                  type="text" 
                  placeholder="e.g. Draft custom telehealth server pricing by next Monday"
                  value={newLog.nextAction}
                  onChange={(e) => setNewLog({...newLog, nextAction: e.target.value})}
                  className="w-full bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-800 font-semibold"
                />
              </div>

              <div>
                <label className="block text-slate-500 font-bold mb-1">On-site Encounter Notes</label>
                <textarea 
                  placeholder="Detail critical client observations, computer cart power issues, scanner evaluations, and clinical feedback..."
                  value={newLog.notes}
                  onChange={(e) => setNewLog({...newLog, notes: e.target.value})}
                  className="w-full h-24 bg-slate-50 border border-slate-200 rounded-lg p-2.5 focus:outline-none focus:border-blue-500 text-slate-800 font-sans font-medium"
                />
              </div>

              <button 
                type="submit"
                className="w-full bg-blue-600 hover:bg-blue-700 text-white font-extrabold py-3 rounded-lg text-xs mt-2 transition shadow-sm shadow-blue-100"
              >
                Append to Active Sales Ledger
              </button>
            </form>
          </div>
        </div>
      )}

    </div>
  );
}

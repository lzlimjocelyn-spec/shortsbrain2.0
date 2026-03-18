import React, { useState, useEffect, useMemo, useCallback } from 'react';
import {
  Globe, RefreshCcw, Brain, MapPin, Sparkles, Download,
  Layers, Zap, Play, TrendingUp, Users,
  Target, ZapOff,
  UploadCloud, ClipboardCheck, Flag,
  Wand2, Palette, Component as ComponentIcon,
  ChevronDown, ChevronUp, FolderKanban, Lightbulb, Copy, Edit2, Save, Plus,
  RotateCcw, Binary, Power, Settings2, Trash2,
  Database, Clock, CheckCircle2, AlertCircle, Loader2,
  Menu, X, Filter, BarChart3, Calendar
} from 'lucide-react';

// --- FIREBASE INTEGRATION ---
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, collection, doc, setDoc, deleteDoc, onSnapshot 
} from 'firebase/firestore';
import { 
  getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged 
} from 'firebase/auth';

// --- 1. CONFIGURATION & CONSTANTS ---
const M_TYPES = ['DAU-SCT', 'DAC-SCT', 'GenAI DAU-SCT', 'Impressions'];
const MARKET_SEGMENTS = ['India', 'Indonesia', 'Japan', 'South Korea', 'AUNZ'];
const MARKET_KEYS = { 'India': 'IN', 'Indonesia': 'ID', 'Japan': 'JP', 'South Korea': 'KR', 'AUNZ': 'AUNZ' };
const MARKET_KEYS_REV = { 'IN': 'India', 'ID': 'Indonesia', 'JP': 'Japan', 'KR': 'South Korea', 'AUNZ': 'AUNZ' };
const AO_CATEGORIES = ['SSC', 'Shelf', 'UTS', 'MVR', 'UTS SFV'];

const GENDERS_KEYS = ['female', 'male', 'total'];
const GENDERS_DISPLAY_MAP = { 'female': 'FEMALE', 'male': 'MALE', 'total': 'GenPop' };
const AGE_BUCKETS = ['18-24', '25-34', '18-34', '35+', 'total'];

// RENAMED: 35+ to 35-44 for display
const AGE_BUCKETS_DISPLAY_MAP = { 
  '18-24': '18-24', 
  '25-34': '25-34', 
  '18-34': '18-34', 
  '35+': '35-44', 
  'total': 'GenPop' 
};

const OKR_TARGETS = {
  'APAC': 0.15, 'INDIA': 0.16, 'INDONESIA': 0.29, 'JAPAN': 1.2, 'SOUTH KOREA': 1.08, 'AUNZ': 1.56,
  'IN': 0.16, 'ID': 0.29, 'JP': 1.2, 'KR': 1.08
};

const NAV_ITEMS = [
  { id: 'Upload', label: 'Data Ingestion', icon: UploadCloud },
  { id: 'OKR', label: 'Shorts OKR Performance', icon: Target },
  { id: 'Global Hub', label: 'Global Holdback', icon: Globe },
  { id: 'Market Hub', label: 'Campaign Holdback', icon: Layers },
];

const CAMPAIGN_CHILDREN = [
  { id: 'AlwaysOn', label: 'Always-On', icon: Zap },
  { id: 'ScaledCreation', label: 'Scaled Creation', icon: Sparkles },
  { id: 'Trends', label: 'Trends', icon: TrendingUp },
  { id: 'CultMo', label: 'CultMo', icon: ComponentIcon },
  { id: 'ArtMo', label: 'ArtMo', icon: Palette },
  { id: 'GenAI Hub', label: 'GenAI Hub', icon: Wand2 }
];

const DEMO_PERMS = [
  { g: 'total', a: 'total', label: 'GenPop' },
  { g: 'total', a: '18-34', label: 'GenPop 18-34' },
  { g: 'male', a: 'total', label: 'Male GenPop' },
  { g: 'female', a: 'total', label: 'Female GenPop' },
];

// --- 2. GLOBAL UTILITIES ---

const cleanStr = (s) => (s || '').toString().replace(/['"]/g, '').replace(/\u00A0/g, ' ').trim();
const superClean = (s) => cleanStr(s).toUpperCase().replace(/[^A-Z0-9]/g, '');
const eq = (a, b) => superClean(a) === superClean(b);

const resolveMarketCode = (m) => {
  const s = cleanStr(m).toUpperCase();
  if (s === 'INDIA' || s === 'IN') return 'IN';
  if (s === 'INDONESIA' || s === 'ID') return 'ID';
  if (s === 'JAPAN' || s === 'JP') return 'JP';
  if (s === 'SOUTH KOREA' || s === 'KR' || s === 'KOREA') return 'KR';
  if (s === 'AUNZ' || s === 'AUSTRALIA' || s === 'NZ') return 'AUNZ';
  return s;
};

// Fixed to show 0.00 explicitly for numeric zeros
const formatCompactNumber = (val) => {
  if (val === 0) return '0.00';
  if (val === null || val === undefined || isNaN(val)) return '-';
  return new Intl.NumberFormat('en-US', {
    notation: 'compact',
    maximumFractionDigits: 1
  }).format(val);
};

const getWeekId = (dateStr) => {
  const d = dateStr ? new Date(dateStr) : new Date();
  if (isNaN(d.getTime())) return `SNAPSHOT-${Date.now()}`;
  const year = d.getFullYear();
  const oneJan = new Date(year, 0, 1);
  const numberOfDays = Math.floor((d - oneJan) / (24 * 60 * 60 * 1000));
  const result = Math.ceil((d.getDay() + 1 + numberOfDays) / 7);
  return `${year}-W${result.toString().padStart(2, '0')}`;
};

const robustParseDate = (dateStr) => {
  const d = cleanStr(dateStr);
  if (!d || d === '-' || d === 'Unknown') return null;
  try {
    if (d.includes('-') && d.split('-')[0].length === 4) return d;
    const parts = d.split(/[-/]/);
    if (parts.length === 3) {
      let v1 = parseInt(parts[0], 10), v2 = parseInt(parts[1], 10), y = parseInt(parts[2], 10);
      if (y < 100) y += 2000;
      let month, day;
      if (v1 > 12) { day = v1; month = v2; }
      else if (v2 > 12) { month = v1; day = v2; }
      else { day = v1; month = v2; }
      if (month > 12) return null;
      return `${y}-${month.toString().padStart(2, '0')}-${day.toString().padStart(2, '0')}`;
    }
    const date = new Date(d);
    return isNaN(date.getTime()) ? null : date.toISOString().split('T')[0];
  } catch { return null; }
};

const calcDaysLive = (startStr, endStr) => {
  const start = robustParseDate(startStr);
  const end = robustParseDate(endStr); 
  if (!start || !end) return 0;
  try {
    const s = new Date(start), e = new Date(end);
    const diffTime = e.getTime() - s.getTime();
    const diffDays = Math.floor(diffTime / (1000 * 60 * 60 * 24));
    return diffDays >= 0 ? diffDays + 1 : 0;
  } catch { return 0; }
};

const isCampaignEnded = (optEndDateStr, campaignEndDateStr) => {
  const optDate = robustParseDate(optEndDateStr);
  const campDate = robustParseDate(campaignEndDateStr);
  if (!optDate || !campDate) return false;
  try { return new Date(optDate) >= new Date(campDate); } catch { return false; }
};

const splitCSVLine = (line) => {
  const columns = [];
  let current = "", inQuotes = false;
  for (let i = 0; i < line.length; i++) {
    const char = line[i];
    if (char === '"') inQuotes = !inQuotes;
    else if (char === ',' && !inQuotes) { columns.push(current.trim()); current = ""; }
    else { current += char; }
  }
  columns.push(current.trim());
  return columns;
};

const findHeader = (headers, targets) => {
  const upperHeaders = headers.map(h => (h || '').toUpperCase().replace(/[^A-Z0-9]/g, ''));
  const targetUpper = targets.map(t => t.toUpperCase().replace(/[^A-Z0-9]/g, ''));
  for (const target of targetUpper) {
    const idx = upperHeaders.indexOf(target);
    if (idx !== -1) return idx;
  }
  return upperHeaders.findIndex(h => targetUpper.some(t => h.includes(t)));
};

const findMetadata = (rowKey, metaMap, marketContext = null) => {
  const campKey = superClean(rowKey);
  if (marketContext) {
    const mktKey = superClean(marketContext);
    if (metaMap[mktKey]?.[campKey]) return metaMap[mktKey][campKey];
  }
  for (const m in metaMap) {
    if (metaMap[m][campKey]) return metaMap[m][campKey];
  }
  return {};
};

const parseCSVData = (text, existingAcc = {}, metaMap = {}, searchPriority = ['Campaign', 'Campaign Name', 'Country', 'Market'], forceAbs = false, marketContext = null) => {
  try {
    const lines = text.split(/\r?\n/).filter(line => line.trim() !== '');
    if (lines.length < 2) return existingAcc;
    const headers = splitCSVLine(lines[0]);
    const identifierIdx = findHeader(headers, searchPriority);
    const valTypeIdx = findHeader(headers, ['Value Type', 'Metric Type']);
    const sliceIdx = findHeader(headers, ['Slice']);
    const dateIdx = findHeader(headers, ['Date', 'Reporting Date', 'Day']);
    const dataStartIdx = findHeader(headers, ['Start Date', 'Trend Start', 'Trend Start Date']);
    const dataEndIdx = findHeader(headers, ['End Date', 'Trend End', 'Trend End Date']);

    if (identifierIdx === -1 || (valTypeIdx === -1 && !forceAbs)) return existingAcc;
    const acc = { ...existingAcc };

    lines.slice(1).forEach(line => {
      const columns = splitCSVLine(line);
      const rowValTypeRaw = valTypeIdx !== -1 ? (columns[valTypeIdx] || '').replace(/['"]/g, '').trim().toUpperCase() : '';
      const rowSlice = sliceIdx !== -1 ? (columns[sliceIdx] || '').replace(/['"]/g, '').trim().toUpperCase() : '';
      const rowDate = dateIdx !== -1 ? robustParseDate(columns[dateIdx]) : null;

      const isRatioRow = !forceAbs && (rowValTypeRaw === 'RATIO (%)' || rowValTypeRaw === 'RATIO' || rowValTypeRaw.includes('LIFT')) && (rowSlice === 'CONTROL' || rowSlice === '' || rowSlice === 'TOTAL');
      const isAbsRow = forceAbs && (rowValTypeRaw === 'DELTA' || rowValTypeRaw === '' || rowValTypeRaw === 'TOTAL') && (rowSlice === 'CONTROL' || rowSlice === '' || rowSlice === 'TOTAL');
      const isSigRow = rowValTypeRaw.includes('TREND FAVORABILITY') && (rowSlice === 'CONTROL' || rowSlice === '' || rowSlice === 'TOTAL');

      if (!isRatioRow && !isSigRow && !isAbsRow) return;

      const rowKey = cleanStr(columns[identifierIdx]) || 'Unknown';
      let gender = 'total';
      const gIdx = findHeader(headers, ['Gender']);
      if (gIdx !== -1) {
        const rawG = (columns[gIdx] || '').toLowerCase().trim();
        if (rawG === 'female' || rawG === 'f') gender = 'female';
        else if (rawG === 'male' || rawG === 'm') gender = 'male';
        else if (rawG === 'total' || rawG === 'genpop' || rawG === 'all' || rawG === '') gender = 'total';
      }
      let age = 'total';
      const aIdx = findHeader(headers, ['Age', 'Age Group']);
      if (aIdx !== -1) {
        const rawA = (columns[aIdx] || '').toLowerCase().trim();
        if (rawA.includes('18-24')) age = '18-24';
        else if (rawA.includes('25-34')) age = '25-34';
        else if (rawA.includes('18-34')) age = '18-34';
        else if (rawA.includes('35')) age = '35+';
        else if (rawA === 'total' || rawA === 'genpop' || rawA === 'all' || rawA === '') age = 'total';
      }

      if (!acc[rowKey]) {
        const meta = findMetadata(rowKey, metaMap, marketContext);
        const startVal = meta.campaignStartDate || (dataStartIdx !== -1 ? robustParseDate(columns[dataStartIdx]) : null);
        const endVal = meta.campaignEndDate || (dataEndIdx !== -1 ? robustParseDate(columns[dataEndIdx]) : null);

        acc[rowKey] = {
          country: rowKey,
          metrics: {},
          isAnchor: superClean(rowKey).includes('GLOBALHOLDBACK'),
          campaignStartDate: startVal,
          campaignEndDate: endVal,
          optimisationEndDate: meta.optimisationEndDate || rowDate, 
          segmentTag: meta.subTab || 'Campaign Hub',
          meta: meta
        };
        M_TYPES.forEach(m => {
          acc[rowKey].metrics[m] = { female: {}, male: {}, total: {} };
          GENDERS_KEYS.forEach(g => {
            AGE_BUCKETS.forEach(a => acc[rowKey].metrics[m][g][a] = { v: 0, sig: 0, abs: 0, isPaused: false, launchDate: null });
          });
        });
      }

      if (rowDate && (!acc[rowKey].optimisationEndDate || rowDate > acc[rowKey].optimisationEndDate)) {
          acc[rowKey].optimisationEndDate = rowDate;
      }

      M_TYPES.forEach(m => {
        const aliases = {
          'DAU-SCT': ['DAU-SCT', 'DAILY SHORTS CREATION TOOL ACTIVE USERS'],
          'DAC-SCT': ['DAC-SCT', 'DAILY SHORTS CONVERTERS'],
          'GenAI DAU-SCT': ['GENAI DAU', 'GENAI DAILY ACTIVE USERS'],
          'Impressions': ['IMPRESSIONS', 'TOTAL IMPRESSIONS', 'REACH']
        };
        const targetCol = headers.findIndex(h => {
          const hUpper = h.toUpperCase();
          const matchAlias = (aliases[m] || []).some(alias => hUpper.includes(alias));
          return matchAlias && !(hUpper.includes('CONFIDENCE') || hUpper.includes('BOUND'));
        });

        if (targetCol === -1) return;
        const rawCell = (columns[targetCol] || '').replace(/['"]/g, '').trim();
        const numericVal = parseFloat(rawCell.replace(/[^\d.-]/g, '')) || 0;

        if (isRatioRow) acc[rowKey].metrics[m][gender][age].v = numericVal;
        else if (isAbsRow) {
          if (m === 'Impressions') acc[rowKey].metrics[m][gender][age].v = numericVal;
          else acc[rowKey].metrics[m][gender][age].abs = numericVal;
        }
        else if (isSigRow) {
          const sigText = rawCell.toUpperCase();
          acc[rowKey].metrics[m][gender][age].sig = (sigText.includes('POSITIVE') || sigText.includes('SSP')) ? 1 : (sigText.includes('NEGATIVE') || sigText.includes('SSN') ? -1 : 0);
        }
      });
    });
    return acc;
  } catch { return existingAcc; }
};

// --- 3. UI VIEW COMPONENTS ---

const HubRowV2 = ({ type, title, icon: Icon, tag, uploadedFiles, handleFileUpload }) => {
  const isPct = type === 'pct';
  const bgColor = isPct ? 'bg-[#1a1500]' : 'bg-[#0a0a0a]';
  const borderColor = isPct ? 'border-amber-500/30' : 'border-blue-500/30';
  const iconColor = isPct ? 'text-amber-500' : 'text-blue-500';
  const iconBg = isPct ? 'bg-amber-500/20' : 'bg-blue-500/20';

  return (
    <div className={`p-6 rounded-lg border ${borderColor} ${bgColor} mb-6 transition-all shadow-xl`}>
      <div className="flex items-center gap-4 mb-6 px-4">
        <div className={`p-2 rounded-lg flex items-center justify-center ${iconBg} ${iconColor}`}>
          <Icon className="w-5 h-5" />
        </div>
        <div className="text-left">
          <h2 className={`text-lg font-bold uppercase tracking-tight ${iconColor}`}>{title}</h2>
          <p className="text-[8px] font-bold text-[#808080] uppercase tracking-[0.3em]">{tag}</p>
        </div>
      </div>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 text-left items-stretch">
        <div className="group relative border border-[#3a3a3a] bg-black rounded-lg p-5 flex flex-col items-center hover:border-[#555] transition-all justify-center text-center">
          <input type="file" className="absolute inset-0 opacity-0 cursor-pointer z-20" onChange={(e) => handleFileUpload(type, 'global', e.target.files[0])} />
          <div className={`w-12 h-12 rounded-lg mb-4 flex items-center justify-center transition-all ${uploadedFiles[type].global ? 'bg-emerald-500/10 text-emerald-400' : 'bg-[#1a1a1a] text-[#555]'}`}>
            <Globe className="w-7 h-7" />
          </div>
          <h3 className="font-bold text-[10px] mb-1.5 uppercase tracking-wider text-[#e0e0e0]">Global Hub Master</h3>
          <div className="text-[8px] font-mono truncate w-full px-2 py-1.5 rounded bg-black border border-[#3a3a3a] text-[#808080] mb-2 text-center">
            {uploadedFiles[type].global ? uploadedFiles[type].global.name : 'PUSH_MASTER_FILE'}
          </div>
        </div>
        <div className="group relative border border-[#3a3a3a] bg-black rounded-lg p-5 flex flex-col items-center hover:border-[#555] transition-all justify-center">
          <div className={`w-12 h-12 rounded-lg mb-4 flex items-center justify-center transition-all ${Object.keys(uploadedFiles[type].countryHB).length > 0 ? 'bg-emerald-500/10 text-emerald-400' : 'bg-[#1a1a1a] text-[#555]'}`}>
            <Flag className="w-7 h-7" />
          </div>
          <h3 className="font-bold text-[10px] mb-3 uppercase tracking-wider text-[#e0e0e0]">Market Hub Nodes</h3>
          <div className="w-full grid grid-cols-5 gap-1 px-1">
            {MARKET_SEGMENTS.map(m => (
              <div key={m} className="relative aspect-square group/item">
                <input type="file" className="absolute inset-0 opacity-0 cursor-pointer z-20" onChange={(e) => handleFileUpload(type, 'countryHB', e.target.files[0], m)} />
                <div className={`w-full h-full rounded-lg border flex items-center justify-center transition-all ${uploadedFiles[type].countryHB[m] ? 'bg-emerald-500/10 border-emerald-500/30 text-emerald-400' : 'bg-black border-[#3a3a3a] text-[#555] hover:border-[#808080]'}`}>
                  <span className="text-[7px] font-black uppercase">{MARKET_KEYS[m]}</span>
                </div>
              </div>
            ))}
          </div>
        </div>
        <div className="group relative border border-[#3a3a3a] bg-black rounded-lg p-5 flex flex-col items-center hover:border-[#555] transition-all justify-center">
          <div className={`w-12 h-12 rounded-lg mb-4 flex items-center justify-center transition-all ${Object.keys(uploadedFiles[type].alwaysOn).length > 0 ? 'bg-emerald-500/10 text-emerald-400' : 'bg-[#1a1a1a] text-[#555]'}`}>
            <Zap className="w-7 h-7" />
          </div>
          <h3 className="font-bold text-[10px] mb-3 uppercase tracking-wider text-[#e0e0e0]">Always-On Trends</h3>
          <div className="w-full grid grid-cols-2 gap-1.5 px-2">
            {AO_CATEGORIES.map(cat => (
              <div key={cat} className="relative h-7 group/item">
                <input type="file" className="absolute inset-0 opacity-0 cursor-pointer z-20" onChange={(e) => handleFileUpload(type, 'alwaysOn', e.target.files[0], cat)} />
                <div className={`w-full h-full rounded-lg border flex items-center justify-center transition-all ${uploadedFiles[type].alwaysOn[cat] ? 'bg-emerald-500/10 border-emerald-500/30 text-emerald-400' : 'bg-black border-[#3a3a3a] text-[#555] hover:border-[#808080]'}`}>
                  <span className="text-[7px] font-black uppercase">{cat}</span>
                </div>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
};

const LandingPage = ({ uploadedFiles, handleFileUpload, startAnalysis, isAnalyzing }) => (
  <div className="min-h-screen bg-black relative flex flex-col items-center py-10 px-6 text-[#e0e0e0] overflow-y-auto">
    <div className="max-w-[1500px] w-full z-10 text-center">
      <div className="mb-12">
        <div className="inline-block mb-4">
          <div className="bg-[#FF0000] w-16 h-16 rounded-2xl flex items-center justify-center mx-auto shadow-2xl shadow-red-500/20"><Brain className="text-white w-8 h-8" /></div>
        </div>
        <h1 className="text-4xl font-bold tracking-tighter mb-1 uppercase">Shorts Brain <span className="text-[#FF0000]">2.0</span></h1>
        <p className="text-[#808080] text-[10px] font-bold tracking-[0.4em] uppercase">APAC Marketing Incrementality Hub</p>
      </div>

      <div className="p-6 rounded-lg border border-emerald-500/30 bg-[#0a1a0a] mb-6 transition-all shadow-xl">
        <div className="flex items-center gap-4 mb-6 px-4">
          <div className="p-2 rounded-lg flex items-center justify-center bg-emerald-500/20 text-emerald-500"><Settings2 className="w-5 h-5" /></div>
          <div className="text-left">
            <h2 className="text-lg font-bold uppercase tracking-tight text-emerald-500">Structural Metadata Configuration</h2>
            <p className="text-[8px] font-bold text-[#808080] uppercase tracking-[0.3em]">Campaign Definitions & State Instructions</p>
          </div>
        </div>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 px-4">
          <div className="group relative border border-[#3a3a3a] bg-black rounded-lg p-5 flex items-center gap-6 hover:border-[#555] transition-all">
            <div className={`w-12 h-12 rounded-xl flex items-center justify-center shrink-0 ${uploadedFiles.shared.campaignInfo ? 'bg-emerald-500/10 text-emerald-400' : 'bg-[#1a1a1a] text-[#555]'}`}><ClipboardCheck className="w-6 h-6" /></div>
            <div className="flex-1 text-left min-w-0">
              <h4 className="text-[10px] font-bold uppercase text-[#e0e0e0] mb-1">structural hierarchy</h4>
              <div className="text-[8px] font-mono truncate px-2 py-1.5 rounded bg-black border border-[#3a3a3a] text-[#808080]">{uploadedFiles.shared.campaignInfo ? uploadedFiles.shared.campaignInfo.name : 'PUSH_STRUCTURAL_CSV'}</div>
            </div>
            <input type="file" className="absolute inset-0 opacity-0 cursor-pointer z-20" onChange={(e) => handleFileUpload('shared', 'campaignInfo', e.target.files[0])} />
          </div>
          <div className="group relative border border-[#3a3a3a] bg-black rounded-lg p-5 flex items-center gap-6 hover:border-[#555] transition-all">
            <div className={`w-12 h-12 rounded-xl flex items-center justify-center shrink-0 ${uploadedFiles.shared.pauseRelive ? 'bg-emerald-500/10 text-emerald-400' : 'bg-[#1a1a1a] text-[#555]'}`}><Power className="w-6 h-6" /></div>
            <div className="flex-1 text-left min-w-0">
              <h4 className="text-[10px] font-bold uppercase text-[#e0e0e0] mb-1">State Instructions</h4>
              <div className="text-[8px] font-mono truncate px-2 py-1.5 rounded bg-black border border-[#3a3a3a] text-[#808080]">{uploadedFiles.shared.pauseRelive ? uploadedFiles.shared.pauseRelive.name : 'PUSH_INSTRUCTIONS_CSV'}</div>
            </div>
            <input type="file" className="absolute inset-0 opacity-0 cursor-pointer z-20" onChange={(e) => handleFileUpload('shared', 'pauseRelive', e.target.files[0])} />
          </div>
        </div>
      </div>

      <HubRowV2 type="pct" title="Ratio-Based Analysis" tag="Relative Lift Streams (%)" icon={TrendingUp} uploadedFiles={uploadedFiles} handleFileUpload={handleFileUpload} />
      <HubRowV2 type="abs" title="Volume-Based Analysis" tag="Discrete Delta Streams (Delta)" icon={Binary} uploadedFiles={uploadedFiles} handleFileUpload={handleFileUpload} />

      <div className="p-6 rounded-lg border border-purple-500/30 bg-[#0d0a1a] mb-6 transition-all shadow-xl">
        <div className="flex items-center gap-4 mb-6 px-4">
          <div className="p-2 rounded-lg flex items-center justify-center bg-purple-500/20 text-purple-500"><Target className="w-5 h-5" /></div>
          <div className="text-left">
            <h2 className="text-lg font-bold uppercase tracking-tight text-purple-500">Attribution Analysis</h2>
            <p className="text-[8px] font-bold text-[#808080] uppercase tracking-[0.3em]">Marketing Pressure & Reach Metrics</p>
          </div>
        </div>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 px-4">
          <div className="group relative border border-[#3a3a3a] bg-black rounded-lg p-5 flex items-center gap-6 hover:border-[#555] transition-all">
            <div className={`w-12 h-12 rounded-xl flex items-center justify-center shrink-0 ${uploadedFiles.attribution.impressions ? 'bg-purple-500/10 text-purple-400' : 'bg-[#1a1a1a] text-[#555]'}`}><BarChart3 className="w-6 h-6" /></div>
            <div className="flex-1 text-left min-w-0">
              <h4 className="text-[10px] font-bold uppercase text-[#e0e0e0] mb-1">Impressions CSV</h4>
              <div className="text-[8px] font-mono truncate px-2 py-1.5 rounded bg-black border border-[#3a3a3a] text-[#808080]">{uploadedFiles.attribution.impressions ? uploadedFiles.attribution.impressions.name : 'PUSH_IMPRESSIONS_CSV'}</div>
            </div>
            <input type="file" className="absolute inset-0 opacity-0 cursor-pointer z-20" onChange={(e) => handleFileUpload('attribution', 'impressions', e.target.files[0])} />
          </div>
        </div>
      </div>

      <button type="button" onClick={() => startAnalysis()} disabled={isAnalyzing} className="px-12 py-5 rounded-2xl font-bold text-base bg-[#FF0000] text-white transition-all hover:bg-red-500 flex items-center gap-4 mx-auto uppercase mt-8 border border-white/10 shadow-2xl shadow-red-500/30 active:scale-95 disabled:opacity-50">
        {isAnalyzing ? <RefreshCcw className="w-5 h-5 animate-spin" /> : <Play className="w-5 h-5" />}
        {isAnalyzing ? 'Processing APAC Data Streams...' : 'Execute Intelligent Engine'}
      </button>
    </div>
  </div>
);

const MetricControlHub = ({ activeMetrics, toggleMetric, handleAllToggle, allowedMetrics = M_TYPES }) => (
  <div className="bg-[#1a1a1a] rounded-lg p-4 border border-[#3a3a3a] flex flex-col sm:flex-row items-center justify-between gap-4 mb-6">
    <div className="flex flex-wrap gap-2 bg-black p-1 rounded-lg border border-[#3a3a3a]">
      {allowedMetrics.map(m => (
        <button key={m} type="button" onClick={() => toggleMetric(m)} className={`px-5 py-2.5 rounded-md text-[10px] font-bold tracking-widest uppercase transition-all cursor-pointer ${activeMetrics.includes(m) ? 'bg-[#FF0000] text-white' : 'text-[#808080] hover:text-white'}`}>
          {m}
        </button>
      ))}
    </div>
    <button type="button" onClick={() => handleAllToggle()} className={`px-6 py-2.5 rounded-md text-[10px] font-bold tracking-widest uppercase border transition-all cursor-pointer ${activeMetrics.length === allowedMetrics.length ? 'bg-white text-black border-white' : 'bg-transparent text-[#808080] border-[#3a3a3a] hover:border-[#808080]'}`}>
      {activeMetrics.length === allowedMetrics.length ? 'Selective View' : 'Sync All Metrics'}
    </button>
  </div>
);

const MasterTableView = ({ data, activeMetrics, isCampaignView = false, hideDates = false, isAlwaysOn = false }) => {
  const themes = {
    female: { 1: 'bg-blue-900/40 text-blue-100', 2: 'bg-blue-900/20', 3: 'bg-blue-950/40 text-blue-400' },
    male: { 1: 'bg-purple-900/40 text-purple-100', 2: 'bg-purple-900/20', 3: 'bg-purple-950/40 text-purple-400' },
    total: { 1: 'bg-amber-900/80 text-amber-50', 2: 'bg-amber-800/20', 3: 'bg-amber-950 text-amber-400 font-bold' }
  };

  const activeAges = AGE_BUCKETS;

  if (!data || data.length === 0) return (
    <div className="py-40 text-center flex flex-col items-center justify-center gap-6">
      <div className="p-6 rounded-full bg-[#1a1a1a] border border-[#3a3a3a]"><ZapOff className="w-12 h-12 text-[#3a3a3a] animate-pulse" /></div>
      <p className="text-[#808080] font-bold text-sm uppercase tracking-widest">No Data Available</p>
    </div>
  );

  return (
    <div className="bg-[#1a1a1a] rounded-lg border border-[#3a3a3a] overflow-hidden overflow-x-auto">
      <table className="w-full text-center border-collapse">
        <thead>
          <tr className="text-[11px] font-bold uppercase tracking-widest border-b border-[#3a3a3a]">
            <th rowSpan={3} className="px-8 py-8 text-left border-r border-[#3a3a3a] bg-[#1a1a1a] sticky left-0 z-40 text-white min-w-[300px]">
              {isAlwaysOn ? 'Trend Identifier' : (isCampaignView ? 'Campaign Entity' : 'Country / Market')}
            </th>
            {GENDERS_KEYS.map((g, gi) => (
              <th key={g} colSpan={activeAges.length * activeMetrics.length} className={`py-6 border-white/10 ${themes[g][1]} ${gi < GENDERS_KEYS.length - 1 ? 'border-r-2 border-white/20' : ''}`}>
                <div className="flex items-center justify-center gap-3"><Users className="w-4 h-4 opacity-50" />{GENDERS_DISPLAY_MAP[g]}</div>
              </th>
            ))}
          </tr>
          <tr className="text-[10px] font-bold uppercase tracking-widest border-b border-[#3a3a3a]">
            {GENDERS_KEYS.map((g) => (
              <React.Fragment key={g}>
                {activeAges.map((a, ai) => (
                  <th key={a} colSpan={activeMetrics.length} className={`py-4 transition-colors ${themes[g][2]} ${ai === activeAges.length - 1 && GENDERS_KEYS.indexOf(g) < GENDERS_KEYS.length - 1 ? 'border-r-2 border-white/20' : 'border-r border-white/5'}`}>{AGE_BUCKETS_DISPLAY_MAP[a]}</th>
                ))}
              </React.Fragment>
            ))}
          </tr>
          <tr className="text-[9px] font-bold uppercase tracking-[0.2em] border-b border-[#3a3a3a]">
            {GENDERS_KEYS.map((g) => (
              <React.Fragment key={g}>
                {activeAges.map((a, ai) => (
                  <React.Fragment key={a}>
                    {activeMetrics.map((m, mi) => (
                      <th key={m} className={`py-3 px-3 font-mono ${themes[g][3]} ${ai === activeAges.length - 1 && mi === activeMetrics.length - 1 && GENDERS_KEYS.indexOf(g) < GENDERS_KEYS.length - 1 ? 'border-r-2 border-white/20' : 'border-r border-white/5'}`}>{m.includes('GenAI') ? 'GenAI' : (m === 'Impressions' ? 'Imprs' : m.split('-')[0])}</th>
                    ))}
                  </React.Fragment>
                ))}
              </React.Fragment>
            ))}
          </tr>
        </thead>
        <tbody className="divide-y divide-white/5">
          {data.map((row, ri) => {
            const isAnchorRow = !!row.isAnchor;
            const daysLive = isAlwaysOn 
              ? calcDaysLive(row.campaignStartDate, row.campaignEndDate || row.optimisationEndDate)
              : calcDaysLive(row.campaignStartDate, row.optimisationEndDate);
            
            return (
              <tr key={ri} className={`transition-all duration-200 ${isAnchorRow ? 'bg-white/[0.05]' : 'hover:bg-white/[0.03]'}`}>
                <td className={`px-8 py-5 text-left border-r border-[#3a3a3a] sticky left-0 z-10 bg-[#111] ${isAnchorRow ? 'text-blue-400 font-bold' : 'text-[#e0e0e0]'}`}>
                  <div className="flex flex-col gap-1.5">
                    <span className="font-bold text-[12px] uppercase tracking-tight">{row.country}</span>
                    {!hideDates && !isAnchorRow && (row.campaignStartDate || isAlwaysOn) && (
                      <div className="flex flex-col gap-1.5 mt-2 p-2 rounded bg-black/40 border border-white/5 shadow-inner">
                        <div className="flex items-center gap-2">
                          <Calendar className="w-3 h-3 text-blue-400" />
                          <span className="text-[9px] font-mono tracking-tighter text-[#888]">
                            <span className="font-bold uppercase text-[8px] mr-1">{isAlwaysOn ? 'Trend Start:' : 'Start:'}</span>
                            {row.campaignStartDate || 'N/A'}
                          </span>
                        </div>
                        <div className="flex items-center gap-2">
                          <Calendar className="w-3 h-3 text-amber-400" />
                          <span className="text-[9px] font-mono tracking-tighter text-[#888]">
                            <span className="font-bold uppercase text-[8px] mr-1">{isAlwaysOn ? 'Trend End:' : 'End:'}</span>
                            {row.campaignEndDate || 'Active'}
                          </span>
                        </div>
                        <div className="flex items-center gap-2 mt-1">
                          <Clock className="w-3 h-3 text-emerald-400" />
                          <span className="text-[9px] font-bold tracking-tighter uppercase text-emerald-400">
                            {isAlwaysOn ? 'Trend Days Live:' : 'Days Live:'} {daysLive}
                          </span>
                        </div>
                      </div>
                    )}
                    {isAnchorRow && isCampaignView && (
                      <span className="text-[9px] uppercase tracking-widest text-blue-500 font-black mt-1 italic">Reference Baseline</span>
                    )}
                  </div>
                </td>
                {GENDERS_KEYS.map((g) => (
                  <React.Fragment key={g}>
                    {activeAges.map((a, ai) => (
                      <React.Fragment key={a}>
                        {activeMetrics.map((m) => {
                          const node = row.metrics[m][g][a];
                          const isEnd = ai === activeAges.length - 1 && activeMetrics.indexOf(m) === activeMetrics.length - 1;
                          let style = "text-slate-500 font-medium", bg = "";
                          const showPaused = node.isPaused && !isAnchorRow;

                          if (showPaused) {
                            style = "text-[#808080] font-bold";
                            bg = "bg-[#1a1a1a]";
                          }
                          else if (node.sig === -1) { style = "text-red-500 font-bold"; bg = "bg-red-500/10"; }
                          else if (node.sig === 1) { style = "text-emerald-500 font-bold"; bg = "bg-emerald-500/10"; }
                          else if (node.v !== 0) { style = "text-slate-100 font-bold"; }

                          return (
                            <td key={m} className={`py-5 px-3 font-mono text-[13px] tabular-nums ${style} ${bg} ${isEnd && GENDERS_KEYS.indexOf(g) < GENDERS_KEYS.length - 1 ? 'border-r-2 border-white/20' : 'border-r border-white/5'}`}>
                              <div className="flex flex-col items-center text-center">
                                {showPaused ? (
                                  <>
                                    <span className="leading-none uppercase">Paused</span>
                                    <span className="text-[7px] opacity-60 font-sans tracking-tight block mt-0.5 font-normal leading-none uppercase italic">
                                      {node.launchDate || 'No Data'}
                                    </span>
                                  </>
                                ) : (
                                  <>
                                    <span>
                                      {m === 'Impressions' 
                                        ? formatCompactNumber(node.v)
                                        : (node.v === 0 ? '0.00' : (node.v > 0 ? `+${node.v.toFixed(2)}` : `${node.v.toFixed(2)}`))}
                                    </span>
                                    {node.abs !== 0 && m !== 'Impressions' && (
                                      <span className="text-[9px] opacity-50 font-sans tracking-tighter block mt-0.5 font-normal leading-none">
                                        ({node.abs > 0 ? `+${Math.round(node.abs).toLocaleString()}` : Math.round(node.abs).toLocaleString()})
                                      </span>
                                    )}
                                  </>
                                )}
                              </div>
                            </td>
                          );
                        })}
                      </React.Fragment>
                    ))}
                  </React.Fragment>
                ))}
              </tr>
            );
          })}
        </tbody>
      </table>
    </div>
  );
};

const OKRAndRecsView = ({ globalData, regionalData, latestDate }) => {
  const [editingId, setEditingId] = useState(null);
  const [editedRows, setEditedRows] = useState({});
  const [manualPointers, setManualPointers] = useState([]);
  const [isAddingManual, setIsAddingManual] = useState(false);
  const [deletedRowIds, setDeletedRowIds] = useState(new Set());
  const [copyFeedback, setCopyFeedback] = useState(null);
  const [newManualForm, setNewManualForm] = useState({ country: 'APAC', campaign: '', age: 'GenPop', gender: 'GenPop', recommendation: 'MAINTAIN', justification: '' });

  const showFeedback = (msg) => { setCopyFeedback(msg); setTimeout(() => setCopyFeedback(null), 2000); };
  const daysLeft = useMemo(() => {
    if (!latestDate) return "TBD";
    const endQ = new Date("2026-03-31");
    const current = new Date(latestDate);
    const diffTime = Math.ceil((endQ - current) / (1000 * 60 * 60 * 24));
    return diffTime > 0 ? diffTime : 0;
  }, [latestDate]);

  const okrStats = useMemo(() => {
    return ['APAC', 'India', 'Indonesia', 'Japan', 'South Korea', 'AUNZ'].map(mName => {
      const record = globalData.find(d => eq(d.country, mName) || eq(d.country, MARKET_KEYS[mName]));
      const actual = record?.metrics?.['DAU-SCT']?.total?.total?.v || 0;
      const target = OKR_TARGETS[mName.toUpperCase()] || 1.00;
      const perfIndex = target > 0 ? (actual / target) * 100 : 0;
      return { market: mName.toUpperCase(), actual, target, perfIndex, isOffline: !record };
    });
  }, [globalData]);

  const recommendationRows = useMemo(() => {
    const tableData = [];
    MARKET_SEGMENTS.forEach(market => {
      const allCampsInMarket = regionalData[market] || [];
      allCampsInMarket.forEach((camp, ci) => {
        if (isCampaignEnded(camp.optimisationEndDate, camp.campaignEndDate)) return;
        const metrics = camp.metrics?.['DAU-SCT'] || {};
        if (metrics.total?.total?.isPaused) return;

        const isNeg = (g, a) => (metrics[g]?.[a]?.v || 0) < -0.0001 && !metrics[g]?.[a]?.isPaused;
        const isPosSig = (g, a) => (metrics[g]?.[a]?.sig || 0) === 1 && !metrics[g]?.[a]?.isPaused;

        const mKey = MARKET_KEYS[market] || market.toUpperCase();
        const aoScalingRestricted = ['SHELF', 'SSC', 'UTS', 'MVR'];
        
        let isAO = false;
        const currentCampaignSuper = superClean(camp.country);
        for (const cat of aoScalingRestricted) {
          if (cat === 'UTS' && currentCampaignSuper.includes('UTSSFV')) continue;
          if (currentCampaignSuper.includes(superClean(cat))) { isAO = true; break; }
        }

        const gpNode = metrics.total?.total || { v: 0, sig: 0 };
        if (!isAO && gpNode.sig === 1 && gpNode.v > 0.001) {
          tableData.push({ id: `CAMP_${market}_${ci}_SC`, country: mKey, campaign: camp.country, age: "GenPop", gender: "GenPop", recommendation: "SCALE", justification: `${mKey} ${camp.country} - Scale GenPop: Stat-sig positive lift (+${gpNode.v.toFixed(2)}%) observed.` });
        }

        const getPausedAges = (gK) => {
          let dirs = [];
          if (isNeg(gK, '18-24') && isNeg(gK, '25-34') && isNeg(gK, '35+')) return ['total'];
          if (isNeg(gK, '18-24') && isNeg(gK, '25-34')) dirs.push('18-34');
          else { if (isNeg(gK, '18-24')) dirs.push('18-24'); if (isNeg(gK, '25-34')) dirs.push('25-34'); }
          if (isNeg(gK, '35+')) dirs.push('35+');
          return dirs;
        };

        const common = getPausedAges('male').filter(a => getPausedAges('female').includes(a));
        const maleUnq = getPausedAges('male').filter(a => !common.includes(a));
        const femaleUnq = getPausedAges('female').filter(a => !common.includes(a));

        const addPause = (gK, aK, tag) => {
          const v = metrics[gK]?.[aK]?.v || 0;
          tableData.push({ id: `CAMP_${market}_${ci}_P_${gK}_${aK}`, country: mKey, campaign: camp.country, age: aK === 'total' ? 'GenPop' : aK, gender: gK === 'total' ? 'GenPop' : gK.toUpperCase(), recommendation: "PAUSE", justification: `${mKey} ${camp.country} - Pause ${tag}${aK} given negative lift (${v.toFixed(2)}%)` });
        };

        common.forEach(a => addPause('total', a, 'G'));
        maleUnq.forEach(a => addPause('male', a, 'M'));
        femaleUnq.forEach(a => addPause('female', a, 'F'));
        
        if (!isAO) {
          DEMO_PERMS.filter(d => d.g !== 'total').forEach(demo => {
            if (isPosSig(demo.g, demo.a)) {
              const v = metrics[demo.g]?.[demo.a]?.v || 0;
              tableData.push({ id: `CAMP_${market}_${ci}_SC_${demo.g}_${demo.a}`, country: mKey, campaign: camp.country, age: demo.a === 'total' ? 'GenPop' : demo.a, gender: demo.g.toUpperCase(), recommendation: "SCALE", justification: `${mKey} ${camp.country} - Scale ${demo.g.charAt(0).toUpperCase()}${demo.a} given positive lift (+${v.toFixed(2)}%)` });
            }
          });
        }
      });
    });
    const merged = [...tableData, ...manualPointers].filter(r => !deletedRowIds.has(r.id));
    return merged.map(r => editedRows[r.id] ? { ...r, ...editedRows[r.id] } : r);
  }, [regionalData, manualPointers, deletedRowIds, editedRows]);

  const handleAddNewManual = () => { if (!newManualForm.campaign) return; setManualPointers(p => [...p, { ...newManualForm, id: `MANUAL_${Date.now()}` }]); setIsAddingManual(false); setNewManualForm({ country: 'APAC', campaign: '', age: 'GenPop', gender: 'GenPop', recommendation: 'MAINTAIN', justification: '' }); };

  return (
    <div className="w-full max-w-[1600px] mx-auto pb-32">
      {copyFeedback && <div className="fixed bottom-10 left-1/2 -translate-x-1/2 z-[100] bg-emerald-500 text-white px-6 py-3 rounded-lg font-bold text-xs uppercase shadow-xl">{copyFeedback}</div>}
      <div className="flex flex-col lg:flex-row justify-between lg:items-end mb-12 gap-8 border-b border-[#3a3a3a] pb-8">
        <div className="space-y-4">
          <h1 className="text-4xl sm:text-5xl font-bold text-white tracking-tight uppercase">Shorts OKR Performance</h1>
          <div className="flex flex-wrap gap-10 pt-4">
            <div className="space-y-1"><p className="text-[10px] font-bold text-[#808080] uppercase tracking-widest">Quarter Start</p><p className="text-lg font-bold text-white">2026-02-01</p></div>
            <div className="space-y-1"><p className="text-[10px] font-bold text-[#808080] uppercase tracking-widest">Reporting Date</p><p className="text-lg font-bold text-emerald-400">{latestDate || "Awaiting Data..."}</p></div>
            <div className="space-y-1"><p className="text-[10px] font-bold text-[#808080] uppercase tracking-widest">Days Left</p><p className="text-lg font-bold text-amber-400">{daysLeft} <span className="text-[10px] text-[#808080] font-normal ml-1">remaining</span></p></div>
          </div>
        </div>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-16">
        {okrStats.map((stat, idx) => {
          const cfg = getStatusConfig(stat.perfIndex, stat.isOffline);
          return (
            <div key={idx} className={`relative ${cfg.cardBg} rounded-lg p-6 border border-[#3a3a3a] transition-all hover:border-[#555] shadow-sm`}>
              <div className="flex justify-between items-start mb-6"><h3 className="text-xl font-bold text-white uppercase">{stat.market}</h3>{!stat.isOffline && <BarChart3 className={`w-5 h-5 ${cfg.color}`} />}</div>
              <div className="flex items-baseline gap-2 mb-4"><span className="text-3xl font-bold text-white">{(stat.perfIndex || 0).toFixed(1)}%</span><span className="text-[9px] font-bold text-[#808080] uppercase tracking-tighter">INDEX</span></div>
              <div className="relative h-1.5 w-full bg-black rounded-full overflow-hidden mb-4"><div className={`h-full ${cfg.accent} transition-all duration-1000`} style={{ width: `${Math.min(stat.perfIndex, 100)}%` }} /></div>
              <div className="flex justify-between pt-4 border-t border-[#3a3a3a] font-mono text-[10px]">
                <div className="text-[#808080] uppercase">Actual: <span className="text-white">+{stat.actual.toFixed(2)}%</span></div>
                <div className="text-[#808080] uppercase">Target: <span className="text-[#b0b0b0]">{stat.target.toFixed(2)}%</span></div>
              </div>
            </div>
          );
        })}
      </div>

      <div className="space-y-8">
        <div className="flex flex-col sm:flex-row items-center justify-between gap-6">
          <div className="flex items-center gap-4"><div className="p-3 bg-[#1a1a1a] rounded-lg border border-[#3a3a3a]"><Lightbulb className="w-6 h-6 text-amber-400" /></div><div><h2 className="text-2xl font-bold text-white uppercase">Strategic Guidance</h2><p className="text-[#808080] text-xs uppercase tracking-widest mt-1 font-medium">Data-Driven Directives & Overrides</p></div></div>
          <div className="flex flex-wrap gap-3">
            <button onClick={() => setIsAddingManual(true)} className="flex items-center gap-2 bg-[#FF0000] text-white px-5 py-2.5 rounded-lg text-[10px] font-bold uppercase hover:bg-red-500 shadow-lg transition-all active:scale-95"><Plus className="w-4 h-4" /> Add Pointer</button>
            <button onClick={() => { if (recommendationRows.length === 0) return; const headers = "Market\tEntity\tAge\tGender\tDirective\tJustification"; const body = recommendationRows.map(r => `${r.country}\t${r.campaign}\t${r.age}\t${r.gender}\t${r.recommendation}\t${r.justification}`).join('\n'); copyToClipboard(`${headers}\n${body}`); showFeedback("Matrix Copied for Sheets"); }} className="flex items-center gap-2 bg-white text-black px-5 py-2.5 rounded-lg text-[10px] font-bold uppercase hover:bg-[#e0e0e0] shadow-lg transition-all active:scale-95"><Copy className="w-4 h-4" /> Copy All</button>
            <button onClick={() => { setDeletedRowIds(new Set()); setEditedRows({}); setManualPointers([]); showFeedback("Matrix Restored"); }} className="flex items-center gap-2 bg-[#1a1a1a] text-[#808080] px-5 py-2.5 rounded-lg text-[10px] font-bold uppercase hover:text-white transition-all border border-[#3a3a3a] active:scale-95"><RotateCcw className="w-4 h-4" /> Restore</button>
          </div>
        </div>

        {isAddingManual && (
          <div className="bg-[#1a1a1a] border border-[#FF0000]/30 rounded-lg p-6 shadow-2xl animate-in fade-in zoom-in duration-200">
            <div className="grid grid-cols-2 md:grid-cols-5 gap-4 mb-6">
              <div><label className="text-[9px] font-bold text-[#808080] uppercase block mb-2">Market</label><input className="w-full bg-black border border-[#3a3a3a] rounded-md p-2.5 text-[11px] font-bold uppercase text-white" value={newManualForm.country} onChange={e => setNewManualForm(p => ({...p, country: e.target.value.toUpperCase()}))} /></div>
              <div><label className="text-[9px] font-bold text-[#808080] uppercase block mb-2">Entity</label><input className="w-full bg-black border border-[#3a3a3a] rounded-md p-2.5 text-[11px] font-bold text-white" placeholder="e.g. Veo" value={newManualForm.campaign} onChange={e => setNewManualForm(p => ({...p, campaign: e.target.value}))} /></div>
              <div><label className="text-[9px] font-bold text-[#808080] uppercase block mb-2">Age</label><input className="w-full bg-black border border-[#3a3a3a] rounded-md p-2.5 text-[11px] font-bold text-white" value={newManualForm.age} onChange={e => setNewManualForm(p => ({...p, age: e.target.value}))} /></div>
              <div><label className="text-[9px] font-bold text-[#808080] uppercase block mb-2">Gender</label><input className="w-full bg-black border border-[#3a3a3a] rounded-md p-2.5 text-[11px] font-bold text-white" value={newManualForm.gender} onChange={e => setNewManualForm(p => ({...p, gender: e.target.value}))} /></div>
              <div><label className="text-[9px] font-bold text-[#808080] uppercase block mb-2">Directive</label>
                <select className="w-full bg-black border border-[#3a3a3a] rounded-md p-2.5 text-[11px] font-bold text-white" value={newManualForm.recommendation} onChange={e => setNewManualForm(p => ({...p, recommendation: e.target.value}))}>
                  <option value="MAINTAIN">MAINTAIN</option><option value="SCALE">SCALE</option><option value="PAUSE">PAUSE</option>
                </select>
              </div>
            </div>
            <textarea className="w-full h-24 bg-black border border-[#3a3a3a] rounded-lg p-3 text-[11px] text-[#b0b0b0] mb-4 resize-none" value={newManualForm.justification} onChange={e => setNewManualForm(p => ({...p, justification: e.target.value}))} placeholder="Context..." />
            <div className="flex justify-end gap-3">
              <button type="button" onClick={() => setIsAddingManual(false)} className="bg-[#1a1a1a] text-[#808080] px-5 py-2.5 rounded-lg font-bold text-[10px] uppercase border border-[#3a3a3a]">Cancel</button>
              <button onClick={handleAddNewManual} className="bg-emerald-600 text-white px-6 py-2.5 rounded-lg font-bold text-[10px] uppercase shadow-lg">Confirm</button>
            </div>
          </div>
        )}

        <div className="bg-[#1a1a1a] rounded-lg border border-[#3a3a3a] overflow-hidden overflow-x-auto shadow-sm">
          <table className="w-full border-collapse text-[11px]">
            <thead>
              <tr className="bg-[#111] text-[#808080] uppercase tracking-widest border-b border-[#3a3a3a] font-bold">
                <th className="px-8 py-6 text-left">Market</th><th className="px-8 py-6 text-left">Entity</th><th className="px-8 py-6 text-center">Age</th><th className="px-8 py-6 text-center">Gender</th><th className="px-8 py-6 text-left">Directive</th><th className="px-8 py-6 text-left">Justification</th><th className="px-8 py-6 text-center">Action</th>
              </tr>
            </thead>
            <tbody className="divide-y divide-white/5">
              {recommendationRows.map(row => (
                <tr key={row.id} className={`hover:bg-white/[0.02] group/row transition-colors ${row.recommendation === 'PAUSE' ? 'bg-red-500/[0.03]' : ''}`}>
                  <td className="px-8 py-4 font-bold uppercase text-blue-400">{row.country}</td>
                  <td className="px-8 py-4 font-bold text-[#e0e0e0] truncate max-w-[200px]">{row.campaign}</td>
                  <td className="px-8 py-4 text-center text-[#b0b0b0] uppercase font-mono">{row.age}</td>
                  <td className="px-8 py-4 text-center text-[#b0b0b0] uppercase font-mono">{row.gender}</td>
                  <td className="px-8 py-4 font-bold">
                    {editingId === row.id ? (
                      <select className="bg-black/60 border border-white/10 rounded-lg p-2 text-[10px] text-white" value={editedRows[row.id]?.recommendation || row.recommendation} onChange={e => setEditedRows(p => ({...p, [row.id]: {...(p[row.id] || row), recommendation: e.target.value}}))}>
                        <option value="MAINTAIN">MAINTAIN</option><option value="SCALE">SCALE</option><option value="PAUSE">PAUSE</option>
                      </select>
                    ) : (
                      <span className={row.recommendation === 'PAUSE' ? 'text-red-400' : (row.recommendation === 'SCALE' ? 'text-emerald-400' : 'text-amber-400')}>{row.recommendation}</span>
                    )}
                  </td>
                  <td className="px-8 py-4 text-[#808080] max-w-[300px] leading-relaxed">
                    {editingId === row.id ? (
                      <textarea className="w-full bg-black/60 border border-white/10 rounded-lg p-2 text-[10px] min-h-[60px] resize-none" value={editedRows[row.id]?.justification || row.justification} onChange={e => setEditedRows(p => ({...p, [row.id]: {...(p[row.id] || row), justification: e.target.value}}))} />
                    ) : row.justification}
                  </td>
                  <td className="px-8 py-5 text-center">
                    <div className="flex items-center justify-center gap-2 opacity-0 group-hover/row:opacity-100 transition-opacity">
                      {editingId === row.id ? (
                        <button onClick={() => setEditingId(null)} className="p-2.5 rounded-xl bg-emerald-600 text-white"><Save className="w-4 h-4" /></button>
                      ) : (
                        <button onClick={() => setEditingId(row.id)} className="p-2.5 rounded-xl bg-white/5 text-slate-400 hover:text-white"><Edit2 className="w-4 h-4" /></button>
                      )}
                      <button onClick={() => { copyToClipboard(`${row.country}\t${row.campaign}\t${row.age}\t${row.gender}\t${row.recommendation}\t${row.justification}`); showFeedback("Row Copied"); }} className="p-2.5 rounded-xl bg-white/5 text-slate-400 hover:text-emerald-400"><Copy className="w-4 h-4" /></button>
                      <button onClick={() => setDeletedRowIds(p => new Set([...p, row.id]))} className="p-2.5 rounded-xl bg-white/5 text-slate-400 hover:text-red-500"><Trash2 className="w-4 h-4" /></button>
                    </div>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
          {recommendationRows.length === 0 && <div className="p-20 text-center text-[#555] font-bold uppercase tracking-widest text-[10px]">Matrix Empty</div>}
        </div>
      </div>
    </div>
  );
};

// --- MAIN APPLICATION COMPONENT ---

const App = () => {
  const [isAnalyzed, setIsAnalyzed] = useState(false);
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [activeTab, setActiveTab] = useState('OKR');
  const [isSidebarOpen, setIsSidebarOpen] = useState(true);
  const [activeMetrics, setActiveMetrics] = useState(['DAU-SCT']);
  const [isCampaignTypeExpanded, setIsCampaignTypeExpanded] = useState(false);
  const [activeMarketSubTab, setActiveMarketSubTab] = useState('India');
  const [latestGlobalDate, setLatestGlobalDate] = useState(null);
  const [user, setUser] = useState(null);

  const [tabMarketFilter, setTabMarketFilter] = useState({ 'ScaledCreation': 'India', 'Trends': 'India', 'CultMo': 'India', 'ArtMo': 'India', 'GenAI Hub': 'India', 'AlwaysOn': 'India' });
  const [subTabFilter, setSubTabFilter] = useState({ 'ScaledCreation': '', 'Trends': '', 'CultMo': '', 'ArtMo': '', 'GenAI Hub': '', 'AlwaysOn': '' });
  const [subSubTabFilter, setSubSubTabFilter] = useState({ 'ScaledCreation': '', 'Trends': '', 'CultMo': '', 'ArtMo': '', 'GenAI Hub': '', 'AlwaysOn': '' });

  const [globalData, setGlobalData] = useState([]);
  const [regionalData, setRegionalData] = useState({});
  const [campaignHubData, setCampaignHubData] = useState({});

  const [uploadedFiles, setUploadedFiles] = useState({
    pct: { global: null, countryHB: {}, alwaysOn: {} },
    abs: { global: null, countryHB: {}, alwaysOn: {} },
    shared: { campaignInfo: null, pauseRelive: null },
    attribution: { impressions: null }
  });

  const [memoryIndex, setMemoryIndex] = useState([]);
  const [memoryStatus, setMemoryStatus] = useState('idle');

  const firebaseConfig = useMemo(() => {
    try {
      return JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
    } catch { return {}; }
  }, []);
  const appId = typeof __app_id !== 'undefined' ? __app_id : 'shorts-brain-v2';

  useEffect(() => {
    if (!firebaseConfig.apiKey) return;
    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const db = getFirestore(app);
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) await signInWithCustomToken(auth, __initial_auth_token);
      else await signInAnonymously(auth);
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, [firebaseConfig]);

  useEffect(() => {
    if (!user) return;
    const db = getFirestore();
    const snapshotsCol = collection(db, 'artifacts', appId, 'public', 'data', 'snapshots');
    const unsubscribe = onSnapshot(snapshotsCol, (snap) => {
      const data = snap.docs.map(doc => ({ ...doc.data(), id: doc.id }));
      setMemoryIndex(data.sort((a, b) => b.timestamp - a.timestamp));
    }, (err) => console.error(err));
    return () => unsubscribe();
  }, [user, appId]);

  const saveSnapshotToCloud = async (snapshotData) => {
    if (!user) return;
    const db = getFirestore();
    setMemoryStatus('saving');
    try {
      const snapId = `snap_${Date.now()}`;
      const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'snapshots', snapId);
      await setDoc(docRef, { ...snapshotData, id: snapId, timestamp: Date.now(), weekId: getWeekId(snapshotData.reportingDate) });
      setMemoryStatus('saved');
      setTimeout(() => setMemoryStatus('idle'), 3000);
    } catch { setMemoryStatus('error'); }
  };

  const startAnalysis = async () => {
    setIsAnalyzing(true);
    try {
      const readFile = (f) => new Promise(res => { if (!f) res(""); const r = new FileReader(); r.onload = e => res(e.target.result); r.readAsText(f); });
      let metaLookup = {};
      if (uploadedFiles.shared.campaignInfo) {
        const text = await readFile(uploadedFiles.shared.campaignInfo);
        const lines = text.split(/\r?\n/).filter(l => l.trim() !== '');
        if (lines.length > 1) {
          const hdrs = splitCSVLine(lines[0]);
          const cIdx = findHeader(hdrs, ['Campaign', 'Campaign Name']), mktIdx = findHeader(hdrs, ['Market', 'Country']), tabIdx = findHeader(hdrs, ['Campaign Tabs', 'Tabs', 'Tab']), subTabIdx = findHeader(hdrs, ['Campaign Sub tabs', 'Sub tabs', 'Sub tab', 'Sub category']), ssTabIdx = findHeader(hdrs, ['Campaign Hub Sub Sub tabs', 'Sub sub tabs']), sIdx = findHeader(hdrs, ['Start Date']), eIdx = findHeader(hdrs, ['End Date']), oIdx = findHeader(hdrs, ['Optimisation End Date']);
          lines.slice(1).forEach(line => {
            const cols = splitCSVLine(line);
            const name = cleanStr(cols[cIdx]);
            if (name) {
              const rawMkt = cleanStr(cols[mktIdx]).toUpperCase();
              const resMkt = MARKET_KEYS_REV[rawMkt] || MARKET_SEGMENTS.find(s => eq(s, rawMkt)) || 'India';
              if (!metaLookup[superClean(resMkt)]) metaLookup[superClean(resMkt)] = {};
              metaLookup[superClean(resMkt)][superClean(name)] = { market: resMkt, tab: cleanStr(cols[tabIdx]), subTab: cleanStr(cols[subTabIdx]), subSubTab: cleanStr(cols[ssTabIdx]), campaignStartDate: cleanStr(cols[sIdx]), campaignEndDate: cleanStr(cols[eIdx]), optimisationEndDate: cleanStr(cols[oIdx]) };
            }
          });
        }
      }
      let instructionMap = {};
      if (uploadedFiles.shared.pauseRelive) {
        const text = await readFile(uploadedFiles.shared.pauseRelive);
        const lines = text.split(/\r?\n/).filter(l => l.trim() !== '');
        if (lines.length > 1) {
          const hdrs = splitCSVLine(lines[0]);
          const cIdx = findHeader(hdrs, ['Campaign', 'Campaign Name']), mIdx = findHeader(hdrs, ['Market', 'Country']), aIdx = findHeader(hdrs, ['Age', 'Age Group', 'Slice']), gIdx = findHeader(hdrs, ['Gender']), iIdx = findHeader(hdrs, ['Instruction', 'Action']), dIdx = findHeader(hdrs, ['Launch Date', 'Date']);
          lines.slice(1).forEach(l => {
            const cols = splitCSVLine(l);
            const mCode = resolveMarketCode(cols[mIdx]);
            const cKey = superClean(cols[cIdx]);
            let aK = cleanStr(cols[aIdx]).toUpperCase();
            aK = (aK === 'GENPOP' || aK === 'TOTAL' || !aK) ? 'total' : aK.toLowerCase().replace(/[^a-z0-9+]/g, '');
            let gK = cleanStr(cols[gIdx]).toUpperCase();
            gK = (gK === 'GENPOP' || gK === 'TOTAL' || !gK) ? 'total' : gK.toLowerCase();
            if (cleanStr(cols[iIdx]).toUpperCase() === 'PAUSE') {
              if (!instructionMap[mCode]) instructionMap[mCode] = {};
              if (!instructionMap[mCode][cKey]) instructionMap[mCode][cKey] = {};
              if (!instructionMap[mCode][cKey][gK]) instructionMap[mCode][cKey][gK] = {};
              instructionMap[mCode][cKey][gK][aK] = cleanStr(cols[dIdx]);
            }
          });
        }
      }

      const structTemp = {};
      const route = (parsedMap, defaultMarket, forceTab = null, forceSub = null) => {
        Object.values(parsedMap).forEach(row => {
          const meta = row.meta || {};
          let tabRaw = forceTab || cleanStr(meta.tab);
          const child = CAMPAIGN_CHILDREN.find(c => eq(c.id, tabRaw) || eq(c.label, tabRaw));
          const mkName = meta.market || defaultMarket || 'India';
          const mCode = resolveMarketCode(mkName);
          const cK = superClean(row.country);
          
          let mInstr = instructionMap[mCode]?.[cK];
          if (!mInstr && meta.subSubTab) mInstr = instructionMap[mCode]?.[superClean(meta.subSubTab)];
          if (!mInstr && meta.subTab) mInstr = instructionMap[mCode]?.[superClean(meta.subTab)];
          if (!mInstr && forceSub) mInstr = instructionMap[mCode]?.[superClean(forceSub)];
          
          const hasEnded = isCampaignEnded(row.optimisationEndDate, row.campaignEndDate);

          M_TYPES.forEach(m => {
            GENDERS_KEYS.forEach(g => {
              AGE_BUCKETS.forEach(a => {
                if (defaultMarket === 'APAC' && !meta.market) return;
                if (hasEnded) { row.metrics[m][g][a].isPaused = true; row.metrics[m][g][a].launchDate = row.campaignEndDate || 'Ended'; return; }
                if (mInstr) {
                  const sA = a.replace(/[^a-z0-9+]/g, ''), pA = (sA === '1824' || sA === '2534') ? '1834' : null;
                  const ks = [[g, sA], pA ? [g, pA] : null, [g, 'total'], ['total', sA], pA ? ['total', pA] : null, ['total', 'total']].filter(Boolean);
                  for (const [tg, ta] of ks) { if (mInstr[tg]?.[ta]) { row.metrics[m][g][a].isPaused = true; row.metrics[m][g][a].launchDate = mInstr[tg][ta]; break; } }
                }
              });
            });
          });
          if (child) {
            const sub = forceSub || cleanStr(meta.subTab) || 'Generic', ss = cleanStr(meta.subSubTab) || 'Default';
            if (!structTemp[child.id]) structTemp[child.id] = {};
            if (!structTemp[child.id][mkName]) structTemp[child.id][mkName] = {};
            if (!structTemp[child.id][mkName][sub]) structTemp[child.id][mkName][sub] = {};
            if (!structTemp[child.id][mkName][sub][ss]) structTemp[child.id][mkName][sub][ss] = {};
            structTemp[child.id][mkName][sub][ss][cK] = row;
          }
        });
      };

      let dGlobalD = null;
      const process = async (st, isAb = false) => {
        const s = uploadedFiles[st];
        let sgd = {};
        if (s.global) {
          const text = await readFile(s.global);
          if (st === 'pct') {
            const lines = text.split(/\r?\n/).filter(l => l.trim() !== '');
            const h = splitCSVLine(lines[0]);
            const di = findHeader(h, ['Date', 'Reporting Date']);
            if (di !== -1) { let md = null; lines.slice(1).forEach(l => { const d = robustParseDate(splitCSVLine(l)[di]); if (d && (!md || d > md)) md = d; }); dGlobalD = md; setLatestGlobalDate(md); }
          }
          sgd = parseCSVData(text, {}, metaLookup, ['Country', 'Market', 'Campaign'], isAb, null);
        }
        const mh = {};
        for (const m of MARKET_SEGMENTS) { if (s.countryHB[m]) mh[m] = parseCSVData(await readFile(s.countryHB[m]), {}, metaLookup, undefined, isAb, m); }
        const ao = {};
        for (const c of AO_CATEGORIES) { if (s.alwaysOn[c]) ao[c] = parseCSVData(await readFile(s.alwaysOn[c]), {}, metaLookup, undefined, isAb, null); }
        return { sgd, mh, ao };
      };

      const pct = await process('pct', false), abs = await process('abs', true);
      let imd = {};
      if (uploadedFiles.attribution.impressions) imd = parseCSVData(await readFile(uploadedFiles.attribution.impressions), {}, metaLookup, ['Campaign', 'Entity', 'Market'], true, null);

      const merge = (pm, am, im = {}) => {
        const res = { ...pm };
        Object.keys(am).forEach(k => { if (res[k]) M_TYPES.forEach(m => { if (m !== 'Impressions') GENDERS_KEYS.forEach(g => { AGE_BUCKETS.forEach(a => { if (am[k].metrics[m]) res[k].metrics[m][g][a].abs = am[k].metrics[m][g][a].abs; }); }); }); });
        Object.keys(im).forEach(k => { if (res[k]) GENDERS_KEYS.forEach(g => { AGE_BUCKETS.forEach(a => { if (im[k].metrics['Impressions']) res[k].metrics['Impressions'][g][a].v = im[k].metrics['Impressions'][g][a].v; }); }); });
        return res;
      };

      const mG = merge(pct.sgd, abs.sgd, imd);
      route(mG, 'APAC');
      const rM = {};
      MARKET_SEGMENTS.forEach(m => {
        const mm = merge(pct.mh[m] || {}, abs.mh[m] || {}, imd);
        rM[m] = Object.values(mm).filter(r => r.meta && (!r.meta.market || eq(r.meta.market, m)));
        route(mm, m);
      });
      AO_CATEGORIES.forEach(c => route(merge(pct.ao[c] || {}, abs.ao[c] || {}, imd), null, 'AlwaysOn', c));

      const fS = {};
      Object.keys(structTemp).forEach(t => {
        fS[t] = {};
        Object.keys(structTemp[t]).forEach(mk => {
          fS[t][mk] = {};
          Object.keys(structTemp[t][mk]).forEach(sub => {
            fS[t][mk][sub] = {};
            Object.keys(structTemp[t][mk][sub]).forEach(ss => { fS[t][mk][sub][ss] = Object.values(structTemp[t][mk][sub][ss]); });
          });
        });
      });

      const arrG = Object.values(mG);
      setGlobalData(arrG); setRegionalData(rM); setCampaignHubData(fS); setIsAnalyzed(true);
      await saveSnapshotToCloud({ reportingDate: dGlobalD, globalData: arrG, regionalData: rM, campaignHubData: fS });
    } catch (err) { console.error(err); } finally { setIsAnalyzing(false); }
  };

  const handleFileUpload = (s, t, f, k) => setUploadedFiles(prev => { 
    const upd = { ...prev[s] }; 
    if (t === 'countryHB' || t === 'alwaysOn') upd[t] = { ...upd[t], [k]: f }; 
    else upd[t] = f; 
    return { ...prev, [s]: upd }; 
  });

  if (!isAnalyzed) return <LandingPage uploadedFiles={uploadedFiles} handleFileUpload={handleFileUpload} startAnalysis={startAnalysis} isAnalyzing={isAnalyzing} />;

  const allowed = activeTab === 'Global Hub' ? M_TYPES.filter(m => m !== 'Impressions') : M_TYPES;
  const filteredMetrics = activeMetrics.filter(m => allowed.includes(m));

  return (
    <div className="flex h-screen bg-black text-[#e0e0e0] overflow-hidden font-sans">
      <aside className={`${isSidebarOpen ? 'w-72' : 'w-20'} transition-all duration-300 bg-[#111] border-r border-[#2a2a2a] flex flex-col z-50`}>
        <div className="p-6 flex items-center gap-3 mb-6 shrink-0 border-b border-[#2a2a2a]">
          <div className="bg-[#FF0000] p-2 rounded-lg flex items-center justify-center shadow-lg shadow-red-500/20"><Brain className="text-white w-5 h-5" /></div>
          {isSidebarOpen && <div><h2 className="text-lg font-bold tracking-tight">BRAIN <span className="text-[#FF0000]">2.0</span></h2><p className="text-[8px] font-bold uppercase text-[#555] tracking-widest">APAC Marketing Hub</p></div>}
        </div>
        <nav className="flex-1 px-4 space-y-1 overflow-y-auto">
          {NAV_ITEMS.map(item => {
            const Icon = item.icon;
            return (
              <button key={item.id} type="button" onClick={() => { if (item.id === 'Upload') setIsAnalyzed(false); else setActiveTab(item.id); }} className={`w-full flex items-center gap-3 p-3 rounded-xl transition-all cursor-pointer ${activeTab === item.id ? 'bg-[#FF0000]/10 text-[#FF0000]' : 'text-[#808080] hover:text-white'}`}>
                <Icon className="w-5 h-5 shrink-0" />{isSidebarOpen && <span className="text-[11px] font-bold uppercase tracking-wider">{item.label}</span>}
              </button>
            );
          })}
          <div className="my-4 border-t border-[#222]"></div>
          <button type="button" onClick={() => setIsCampaignTypeExpanded(!isCampaignTypeExpanded)} className="w-full flex items-center justify-between p-3 rounded-xl text-[#808080] hover:text-white cursor-pointer">
            <div className="flex items-center gap-3"><FolderKanban className="w-5 h-5 shrink-0" />{isSidebarOpen && <span className="text-[11px] font-bold uppercase tracking-widest">Campaign Hub</span>}</div>
            {isSidebarOpen && (isCampaignTypeExpanded ? <ChevronUp className="w-4 h-4" /> : <ChevronDown className="w-4 h-4" />)}
          </button>
          {isCampaignTypeExpanded && isSidebarOpen && (
            <div className="pl-4 space-y-1 animate-in slide-in-from-top-2 duration-300">
              {CAMPAIGN_CHILDREN.map(child => {
                const Icon = child.icon;
                return (
                  <button key={child.id} type="button" onClick={() => setActiveTab(child.id)} className={`w-full flex items-center gap-3 p-3 rounded-lg transition-all cursor-pointer ${activeTab === child.id ? 'bg-[#FF0000]/10 text-[#FF0000]' : 'text-[#555] hover:bg-white/5 hover:text-white'}`}>
                    <Icon className="w-4 h-4" />
                    {child.label}
                  </button>
                );
              })}
            </div>
          )}
          {isSidebarOpen && memoryIndex.length > 0 && (
            <div className="mt-8 pt-4 border-t border-[#222]">
              <div className="flex items-center gap-2 px-3 mb-3"><Database className="w-4 h-4 text-[#444]" /><span className="text-[10px] font-bold uppercase text-[#444] tracking-widest">Memory</span></div>
              <div className="space-y-1 max-h-[300px] overflow-y-auto">
                {memoryIndex.map(snap => (
                  <div key={snap.id} className="w-full flex items-center gap-2 px-3 py-2 rounded-lg text-left transition-all hover:bg-white/5 group">
                    <button type="button" onClick={() => { setGlobalData(snap.globalData || []); setRegionalData(snap.regionalData || {}); setCampaignHubData(snap.campaignHubData || {}); setLatestGlobalDate(snap.reportingDate); setIsAnalyzed(true); }} className="flex-1 text-[10px] font-bold text-[#666] group-hover:text-white">{snap.weekId}</button>
                    <button type="button" onClick={() => { if (user) deleteDoc(doc(getFirestore(), 'artifacts', appId, 'public', 'data', 'snapshots', snap.id)); }} className="opacity-0 group-hover:opacity-100"><Trash2 className="w-3 h-3 text-red-500" /></button>
                  </div>
                ))}
              </div>
            </div>
          )}
        </nav>
        <button type="button" onClick={() => setIsSidebarOpen(!isSidebarOpen)} className="p-6 border-t border-[#2a2a2a] text-[#555] hover:text-white flex items-center justify-center"><Menu className="w-5 h-5" /></button>
      </aside>
      <div className="flex-1 flex flex-col overflow-hidden relative">
        <header className="px-8 py-5 border-b border-[#2a2a2a] flex items-center justify-between bg-[#0a0a0a]/80 backdrop-blur-md sticky top-0 z-40">
          <h4 className="text-sm font-bold text-white uppercase tracking-widest">{activeTab}</h4>
          <div className="flex items-center gap-4"><button type="button" className="bg-white text-black px-6 py-2.5 rounded-lg text-[10px] font-bold uppercase hover:bg-[#e0e0e0] shadow-xl flex items-center gap-2"><Download className="w-3.5 h-3.5" /> Export</button></div>
        </header>
        <main className="flex-1 overflow-auto p-10 relative">
          {activeTab === 'OKR' && <OKRAndRecsView globalData={globalData} regionalData={regionalData} latestDate={latestGlobalDate} />}
          {(activeTab === 'Global Hub' || activeTab === 'Market Hub') && (
            <div className="space-y-8 animate-in fade-in duration-500">
              <MetricControlHub activeMetrics={filteredMetrics} allowedMetrics={allowed} toggleMetric={m => setActiveMetrics(p => p.includes(m) ? (p.length > 1 ? p.filter(x => x !== m) : p) : [...p, m])} handleAllToggle={() => setActiveMetrics(p => p.length === allowed.length ? ['DAU-SCT'] : [...allowed])} />
              {activeTab === 'Market Hub' && (
                <div className="flex items-center gap-4 p-4 bg-[#111] rounded-xl border border-[#2a2a2a] w-fit shadow-lg"><MapPin className="w-6 h-6 text-red-600" /><select value={activeMarketSubTab} onChange={e => setActiveMarketSubTab(e.target.value)} className="bg-transparent text-white font-bold uppercase outline-none cursor-pointer pr-8">{MARKET_SEGMENTS.map(m => <option key={m} value={m} className="bg-neutral-900">{m}</option>)}</select></div>
              )}
              <MasterTableView data={activeTab === 'Global Hub' ? globalData : (() => { const campaigns = regionalData[activeMarketSubTab] || []; const globalRef = globalData.find(d => eq(d.country, activeMarketSubTab) || eq(d.country, MARKET_KEYS[activeMarketSubTab])); return globalRef ? [{ ...globalRef, isAnchor: true }, ...campaigns] : campaigns; })()} activeMetrics={filteredMetrics} isCampaignView={activeTab === 'Market Hub'} hideDates={activeTab === 'Global Hub'} />
            </div>
          )}
          {CAMPAIGN_CHILDREN.some(c => c.id === activeTab) && (
            <div className="space-y-8 animate-in fade-in duration-500">
              <MetricControlHub activeMetrics={filteredMetrics} allowedMetrics={allowed} toggleMetric={m => setActiveMetrics(p => p.includes(m) ? (p.length > 1 ? p.filter(x => x !== m) : p) : [...p, m])} handleAllToggle={() => setActiveMetrics(p => p.length === allowed.length ? ['DAU-SCT'] : [...allowed])} />
              <div className="flex items-center gap-4 p-4 bg-[#111] rounded-xl border border-[#2a2a2a] w-fit shadow-lg"><MapPin className="w-6 h-6 text-red-600" /><select value={tabMarketFilter[activeTab]} onChange={e => setTabMarketFilter(p => ({ ...p, [activeTab]: e.target.value }))} className="bg-transparent text-white font-bold uppercase outline-none cursor-pointer pr-8">{MARKET_SEGMENTS.map(m => <option key={m} value={m} className="bg-neutral-900">{m}</option>)}</select></div>
              <MasterTableView data={(() => { const mkt = tabMarketFilter[activeTab]; const sub = subTabFilter[activeTab] || 'Generic'; const ss = subSubTabFilter[activeTab] || 'Default'; const path = campaignHubData[activeTab]?.[mkt]?.[sub]; const rows = ss === 'Default' || !ss ? (path ? Object.values(path).flat() : []) : (path?.[ss] || []); return activeTab === 'AlwaysOn' ? [...rows].sort((a,b) => (a.campaignStartDate || '').localeCompare(b.campaignStartDate || '')) : rows; })()} activeMetrics={filteredMetrics} isCampaignView isAlwaysOn={activeTab === 'AlwaysOn'} />
            </div>
          )}
        </main>
      </div>
      <style>{`
        ::-webkit-scrollbar { width: 5px; height: 5px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: #2a2a2a; border-radius: 10px; }
        ::-webkit-scrollbar-thumb:hover { background: #444; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        select { appearance: none; background: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 24 24' stroke='white'%3E%3Cpath stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M19 9l-7 7-7-7' /%3E%3C/svg%3E") no-repeat right 0.5rem center; background-size: 1em; }
        .animate-in { animation: fadeIn 0.4s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
      `}</style>
    </div>
  );
};

export default App;

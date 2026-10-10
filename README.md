import React, { useEffect, useRef, useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ScrollView,
  ActivityIndicator,
  SafeAreaView,
  StatusBar,
  Platform,
  StyleSheet,
} from 'react-native';
import { WebView } from 'react-native-webview';
import { File, Paths } from 'expo-file-system';
import * as Sharing from 'expo-sharing';
import * as SecureStore from 'expo-secure-store';
import AsyncStorage from '@react-native-async-storage/async-storage';

/* ───────────── الألوان ───────────── */
const colors = {
  bg: '#0f1115',
  card: '#181b22',
  border: '#2a2f3a',
  text: '#e8eaf0',
  muted: '#9aa3b2',
  primary: '#4f8cff',
  success: '#2fbf71',
  danger: '#e5484d',
};
const rtl = { textAlign: 'right', writingDirection: 'rtl' };

/* ───────────── التخزين ───────────── */
const KEY_NAMES = {
  claude: 'claude_api_key',
  gemini: 'gemini_api_key',
  supabaseUrl: 'supabase_url',
  supabaseKey: 'supabase_anon_key',
};

async function loadAllSecrets() {
  const out = {};
  for (const [k, name] of Object.entries(KEY_NAMES)) {
    try {
      out[k] = (await SecureStore.getItemAsync(name)) || '';
    } catch {
      out[k] = '';
    }
  }
  return out;
}

async function setSecret(name, value) {
  const v = (value || '').trim();
  if (!v) return SecureStore.deleteItemAsync(name);
  return SecureStore.setItemAsync(name, v);
}

async function loadDraft() {
  try {
    return (await AsyncStorage.getItem('code_draft')) || '';
  } catch {
    return '';
  }
}

async function saveDraft(code) {
  try {
    await AsyncStorage.setItem('code_draft', code);
  } catch {}
}

/* ───────────── Claude و Gemini ───────────── */
const CLAUDE_MODEL = 'claude-sonnet-5-5';
const GEMINI_MODEL = 'gemini-2.5-flash';

const SYSTEM = `You are a coding assistant inside a mobile app builder.
The project is a single self-contained HTML file (HTML + CSS + JavaScript together) that is previewed in a WebView.
You receive the current code and a change request written by the user (often in Arabic).
- If the current code is empty, create a complete new project that satisfies the request.
- Otherwise apply only the requested change and keep everything else working.
Reply with the FULL updated file inside one \`\`\`html code block and nothing else (no explanation).`;

function buildPrompt(code, request) {
  return `Current code:\n\`\`\`html\n${code || '(empty)'}\n\`\`\`\n\nRequest:\n${request}`;
}

function extractCode(text) {
  const m = text.match(/```(?:html)?\s*\n([\s\S]*?)```/i);
  return (m ? m[1] : text).trim();
}

async function readJson(res) {
  let data = null;
  try {
    data = await res.json();
  } catch {}
  if (!res.ok) throw new Error(data?.error?.message || `HTTP ${res.status}`);
  return data;
}

async function askClaude(apiKey, code, request) {
  if (!apiKey) throw new Error('أضف مفتاح Claude أولاً من قائمة المفاتيح');
  const res = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'x-api-key': apiKey,
      'anthropic-version': '2023-06-01',
    },
    body: JSON.stringify({
      model: CLAUDE_MODEL,
      max_tokens: 8000,
      system: SYSTEM,
      messages: [{ role: 'user', content: buildPrompt(code, request) }],
    }),
  });
  const data = await readJson(res);
  const text = (data.content || []).map((b) => b.text || '').join('');
  if (!text) throw new Error('رد Claude جاء فارغاً');
  return extractCode(text);
}

async function askGemini(apiKey, code, request) {
  if (!apiKey) throw new Error('أضف مفتاح Gemini أولاً من قائمة المفاتيح');
  const res = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_MODEL}:generateContent`,
    {
      method: 'POST',
      headers: { 'content-type': 'application/json', 'x-goog-api-key': apiKey },
      body: JSON.stringify({
        systemInstruction: { parts: [{ text: SYSTEM }] },
        contents: [{ role: 'user', parts: [{ text: buildPrompt(code, request) }] }],
      }),
    }
  );
  const data = await readJson(res);
  const parts = data?.candidates?.[0]?.content?.parts || [];
  const text = parts.map((p) => p.text || '').join('');
  if (!text) throw new Error('رد Gemini جاء فارغاً');
  return extractCode(text);
}

/* ───────────── Supabase ───────────── */
function sbHeaders(key, extra = {}) {
  return { apikey: key, Authorization: `Bearer ${key}`, 'content-type': 'application/json', ...extra };
}
function sbBase(secrets) {
  if (!secrets.supabaseUrl || !secrets.supabaseKey) {
    throw new Error('أضف رابط Supabase والمفتاح من قائمة المفاتيح');
  }
  return secrets.supabaseUrl.trim().replace(/\/+$/, '');
}
async function sbCheck(res) {
  if (res.ok) return res;
  let msg = `HTTP ${res.status}`;
  try {
    const d = await res.json();
    msg = d.message || d.error || msg;
  } catch {}
  throw new Error(msg);
}

async function saveProject(secrets, name, code) {
  const base = sbBase(secrets);
  const res = await fetch(`${base}/rest/v1/projects?on_conflict=name`, {
    method: 'POST',
    headers: sbHeaders(secrets.supabaseKey, { Prefer: 'resolution=merge-duplicates,return=minimal' }),
    body: JSON.stringify({ name, code, updated_at: new Date().toISOString() }),
  });
  await sbCheck(res);
}

async function listProjects(secrets) {
  const base = sbBase(secrets);
  const res = await fetch(`${base}/rest/v1/projects?select=name,updated_at&order=updated_at.desc`, {
    headers: sbHeaders(secrets.supabaseKey),
  });
  await sbCheck(res);
  return res.json();
}

async function loadProject(secrets, name) {
  const base = sbBase(secrets);
  const res = await fetch(`${base}/rest/v1/projects?name=eq.${encodeURIComponent(name)}&select=code`, {
    headers: sbHeaders(secrets.supabaseKey),
  });
  await sbCheck(res);
  const rows = await res.json();
  return rows?.[0]?.code ?? '';
}

/* ───────────── قائمة المفاتيح ───────────── */
const FIELDS = [
  { id: 'claude', label: 'مفتاح Claude (Anthropic API)', ph: 'sk-ant-...', secure: true },
  { id: 'gemini', label: 'مفتاح Gemini (Google AI Studio)', ph: 'AIza...', secure: true },
  { id: 'supabaseUrl', label: 'رابط مشروع Supabase', ph: 'https://xxxx.supabase.co', secure: false },
  { id: 'supabaseKey', label: 'مفتاح Supabase (anon public)', ph: 'eyJ...', secure: true },
];

function KeysScreen({ secrets, setSecrets }) {
  const [values, setValues] = useState(secrets);
  const [show, setShow] = useState({});
  const [status, setStatus] = useState('');

  useEffect(() => {
    setValues(secrets);
  }, [secrets]);

  async function save() {
    try {
      const clean = {};
      for (const f of FIELDS) {
        const v = (values[f.id] || '').trim();
        clean[f.id] = v;
        await setSecret(KEY_NAMES[f.id], v);
      }
      setSecrets(clean);
      setStatus('تم حفظ المفاتيح بشكل مشفر على جهازك');
    } catch (e) {
      setStatus('تعذر الحفظ: ' + e.message);
    }
  }

  return (
    <ScrollView contentContainerStyle={k.wrap} keyboardShouldPersistTaps="handled">
      <Text style={[k.title, rtl]}>المفاتيح (API)</Text>
      <Text style={[k.hint, rtl]}>المفاتيح تُحفظ مشفرة في جهازك فقط، ولا تُرسل إلا للخدمة الخاصة بها.</Text>

      {FIELDS.map((f) => (
        <View key={f.id} style={k.card}>
          <Text style={[k.label, rtl]}>{f.label}</Text>
          <View style={k.row}>
            <TextInput
              style={k.input}
              value={values[f.id] || ''}
              onChangeText={(t) => setValues({ ...values, [f.id]: t })}
              placeholder={f.ph}
              placeholderTextColor={colors.muted}
              autoCapitalize="none"
              autoCorrect={false}
              secureTextEntry={f.secure && !show[f.id]}
            />
            {f.secure ? (
              <TouchableOpacity onPress={() => setShow({ ...show, [f.id]: !show[f.id] })}>
                <Text style={k.eye}>{show[f.id] ? 'إخفاء' : 'إظهار'}</Text>
              </TouchableOpacity>
            ) : null}
          </View>
          <Text style={[k.state, rtl, { color: secrets[f.id] ? colors.success : colors.muted }]}>
            {secrets[f.id] ? 'محفوظ' : 'غير مضاف'}
          </Text>
        </View>
      ))}

      <TouchableOpacity style={k.btn} onPress={save}>
        <Text style={k.btnText}>حفظ المفاتيح</Text>
      </TouchableOpacity>
      {status ? <Text style={[k.status, rtl]}>{status}</Text> : null}
    </ScrollView>
  );
}

/* ───────────── قائمة الكود ───────────── */
function safeFileName(name) {
  const base = (name || '').trim().replace(/[^\w\u0600-\u06FF-]+/g, '_');
  return (base || 'project') + '.html';
}

function CodeScreen({ secrets, code, setCode, projectName, setProjectName }) {
  const [running, setRunning] = useState(false);
  const [status, setStatus] = useState('');
  const [projects, setProjects] = useState(null);

  async function exportFile() {
    if (!code.trim()) return setStatus('الكود فاضي، ما في شيء للتصدير');
    try {
      const file = new File(Paths.cache, safeFileName(projectName));
      if (file.exists) file.delete();
      file.create();
      file.write(code);
      if (!(await Sharing.isAvailableAsync())) return setStatus('المشاركة غير متاحة على هذا الجهاز');
      await Sharing.shareAsync(file.uri, { mimeType: 'text/html', dialogTitle: 'تصدير الموقع' });
      setStatus('تم تجهيز الملف: ' + safeFileName(projectName));
    } catch (e) {
      setStatus('تعذر التصدير: ' + e.message);
    }
  }

  async function saveCloud() {
    if (!projectName.trim()) return setStatus('اكتب اسم المشروع أولاً');
    try {
      setStatus('جاري الحفظ في Supabase...');
      await saveProject(secrets, projectName.trim(), code);
      setStatus('تم حفظ المشروع في Supabase');
    } catch (e) {
      setStatus('تعذر الحفظ: ' + e.message);
    }
  }

  async function openList() {
    try {
      setStatus('جاري جلب المشاريع...');
      const list = await listProjects(secrets);
      setProjects(list);
      setStatus(list.length ? '' : 'ما في مشاريع محفوظة بعد');
    } catch (e) {
      setStatus('تعذر الجلب: ' + e.message);
    }
  }

  async function openProject(name) {
    try {
      const c = await loadProject(secrets, name);
      setCode(c);
      setProjectName(name);
      setProjects(null);
      setRunning(false);
      setStatus('تم فتح المشروع: ' + name);
    } catch (e) {
      setStatus('تعذر الفتح: ' + e.message);
    }
  }

  return (
    <View style={c.wrap}>
      <View style={c.bar}>
        <TextInput
          style={c.name}
          value={projectName}
          onChangeText={setProjectName}
          placeholder="اسم المشروع"
          placeholderTextColor={colors.muted}
          autoCapitalize="none"
        />
        <TouchableOpacity
          style={[c.run, { backgroundColor: running ? colors.border : colors.success }]}
          onPress={() => setRunning(!running)}
        >
          <Text style={c.runText}>{running ? 'المحرر' : 'تشغيل ▶'}</Text>
        </TouchableOpacity>
      </View>

      {running ? (
        <View style={c.preview}>
          <WebView
            originWhitelist={['*']}
            source={{ html: code || '<h3 style="font-family:sans-serif">الكود فاضي</h3>' }}
            javaScriptEnabled
            domStorageEnabled
            setSupportMultipleWindows={false}
          />
        </View>
      ) : (
        <TextInput
          style={c.editor}
          value={code}
          onChangeText={setCode}
          multiline
          autoCapitalize="none"
          autoCorrect={false}
          spellCheck={false}
          textAlignVertical="top"
          placeholder={'اكتب الكود هنا (HTML كامل)\nأو اذهب لقائمة التعديل واطلب من Claude أو Gemini ينشئه'}
          placeholderTextColor={colors.muted}
        />
      )}

      <View style={c.actions}>
        <TouchableOpacity style={c.act} onPress={exportFile}>
          <Text style={c.actText}>تصدير / تحميل</Text>
        </TouchableOpacity>
        <TouchableOpacity style={c.act} onPress={saveCloud}>
          <Text style={c.actText}>حفظ في Supabase</Text>
        </TouchableOpacity>
        <TouchableOpacity style={c.act} onPress={openList}>
          <Text style={c.actText}>فتح مشروع</Text>
        </TouchableOpacity>
        <TouchableOpacity style={c.act} onPress={() => setCode('')}>
          <Text style={[c.actText, { color: colors.danger }]}>مسح</Text>
        </TouchableOpacity>
      </View>

      {status ? <Text style={[c.status, rtl]}>{status}</Text> : null}

      {projects && projects.length ? (
        <View style={c.list}>
          <ScrollView keyboardShouldPersistTaps="handled">
            {projects.map((p) => (
              <TouchableOpacity key={p.name} style={c.item} onPress={() => openProject(p.name)}>
                <Text style={[c.itemText, rtl]}>{p.name}</Text>
              </TouchableOpacity>
            ))}
          </ScrollView>
          <TouchableOpacity onPress={() => setProjects(null)}>
            <Text style={c.close}>إغلاق</Text>
          </TouchableOpacity>
        </View>
      ) : null}
    </View>
  );
}

/* ───────────── قائمة التعديل ───────────── */
function EditScreen({ secrets, code, setCode, goToCode }) {
  const [engine, setEngine] = useState('claude');
  const [request, setRequest] = useState('');
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState('');
  const [done, setDone] = useState('');
  const [prev, setPrev] = useState(null);

  async function apply() {
    if (!request.trim()) return setError('اكتب التعديل المطلوب أولاً');
    setBusy(true);
    setError('');
    setDone('');
    try {
      const ask = engine === 'claude' ? askClaude : askGemini;
      const newCode = await ask(secrets[engine], code, request.trim());
      setPrev(code);
      setCode(newCode);
      setDone('تم وضع التعديل داخل قائمة الكود');
      setRequest('');
    } catch (e) {
      setError(e.message);
    } finally {
      setBusy(false);
    }
  }

  function undo() {
    if (prev === null) return;
    setCode(prev);
    setPrev(null);
    setDone('تم التراجع عن آخر تعديل');
  }

  return (
    <ScrollView contentContainerStyle={e.wrap} keyboardShouldPersistTaps="handled">
      <Text style={[e.title, rtl]}>التعديل بالذكاء الاصطناعي</Text>
      <Text style={[e.hint, rtl]}>
        اختر Claude أو Gemini واكتب المطلوب، والتعديل ينزل مباشرة في قائمة الكود. لو الكود فاضي سينشئ مشروعاً جديداً.
      </Text>

      <View style={e.engines}>
        {[
          { id: 'claude', label: 'Claude' },
          { id: 'gemini', label: 'Gemini' },
        ].map((x) => (
          <TouchableOpacity
            key={x.id}
            style={[e.engine, engine === x.id && e.engineOn]}
            onPress={() => setEngine(x.id)}
          >
            <Text style={[e.engineText, engine === x.id && { color: '#fff' }]}>{x.label}</Text>
            <Text style={[e.keyState, engine === x.id && { color: '#fff' }]}>
              {secrets[x.id] ? 'المفتاح مضاف' : 'بدون مفتاح'}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      <TextInput
        style={[e.input, rtl]}
        value={request}
        onChangeText={setRequest}
        multiline
        placeholder="مثال: اعمل صفحة لمطعم فيها قائمة الأكل وزر اتصال، بألوان داكنة"
        placeholderTextColor={colors.muted}
        textAlignVertical="top"
      />

      <TouchableOpacity style={[e.btn, busy && { opacity: 0.6 }]} onPress={apply} disabled={busy}>
        {busy ? <ActivityIndicator color="#fff" /> : <Text style={e.btnText}>نفّذ التعديل</Text>}
      </TouchableOpacity>

      {error ? <Text style={[e.error, rtl]}>{error}</Text> : null}

      {done ? (
        <View style={e.doneBox}>
          <Text style={[e.done, rtl]}>{done}</Text>
          <View style={e.doneRow}>
            <TouchableOpacity onPress={goToCode}>
              <Text style={e.link}>افتح الكود</Text>
            </TouchableOpacity>
            {prev !== null ? (
              <TouchableOpacity onPress={undo}>
                <Text style={[e.link, { color: colors.danger }]}>تراجع</Text>
              </TouchableOpacity>
            ) : null}
          </View>
        </View>
      ) : null}
    </ScrollView>
  );
}

/* ───────────── التطبيق ───────────── */
const TABS = [
  { id: 'keys', label: 'المفاتيح' },
  { id: 'code', label: 'الكود' },
  { id: 'edit', label: 'التعديل' },
];

export default function App() {
  const [tab, setTab] = useState('keys');
  const [secrets, setSecrets] = useState({});
  const [code, setCode] = useState('');
  const [projectName, setProjectName] = useState('');
  const loaded = useRef(false);

  useEffect(() => {
    (async () => {
      const s = await loadAllSecrets();
      const d = await loadDraft();
      setSecrets(s);
      setCode(d);
      loaded.current = true;
      if (s.claude || s.gemini) setTab('code');
    })();
  }, []);

  useEffect(() => {
    if (!loaded.current) return;
    const t = setTimeout(() => saveDraft(code), 500);
    return () => clearTimeout(t);
  }, [code]);

  return (
    <SafeAreaView style={a.root}>
      <StatusBar barStyle="light-content" backgroundColor={colors.bg} />
      <View style={a.body}>
        {tab === 'keys' && <KeysScreen secrets={secrets} setSecrets={setSecrets} />}
        {tab === 'code' && (
          <CodeScreen
            secrets={secrets}
            code={code}
            setCode={setCode}
            projectName={projectName}
            setProjectName={setProjectName}
          />
        )}
        {tab === 'edit' && (
          <EditScreen secrets={secrets} code={code} setCode={setCode} goToCode={() => setTab('code')} />
        )}
      </View>

      <View style={a.tabs}>
        {TABS.map((t) => (
          <TouchableOpacity key={t.id} style={a.tab} onPress={() => setTab(t.id)}>
            <Text style={[a.tabText, tab === t.id && a.tabOn]}>{t.label}</Text>
          </TouchableOpacity>
        ))}
      </View>
    </SafeAreaView>
  );
}

/* ───────────── التنسيق ───────────── */
const mono = Platform.select({ ios: 'Menlo', android: 'monospace', default: 'monospace' });
const box = { backgroundColor: colors.card, borderColor: colors.border, borderWidth: 1, borderRadius: 12 };
const primaryBtn = { backgroundColor: colors.primary, borderRadius: 10, padding: 14, alignItems: 'center' };

const a = StyleSheet.create({
  root: {
    flex: 1,
    backgroundColor: colors.bg,
    paddingTop: Platform.OS === 'android' ? StatusBar.currentHeight : 0,
  },
  body: { flex: 1 },
  tabs: { flexDirection: 'row', borderTopColor: colors.border, borderTopWidth: 1, backgroundColor: colors.card },
  tab: { flex: 1, alignItems: 'center', paddingVertical: 14 },
  tabText: { color: colors.muted, fontSize: 15 },
  tabOn: { color: colors.primary, fontWeight: '700' },
});

const k = StyleSheet.create({
  wrap: { padding: 16, paddingBottom: 40 },
  title: { color: colors.text, fontSize: 22, fontWeight: '700', marginBottom: 6 },
  hint: { color: colors.muted, fontSize: 13, marginBottom: 16 },
  card: { ...box, padding: 12, marginBottom: 12 },
  label: { color: colors.text, fontSize: 14, marginBottom: 8 },
  row: { flexDirection: 'row', alignItems: 'center' },
  input: {
    flex: 1,
    color: colors.text,
    borderColor: colors.border,
    borderWidth: 1,
    borderRadius: 8,
    paddingHorizontal: 10,
    paddingVertical: 8,
    textAlign: 'left',
  },
  eye: { color: colors.primary, marginLeft: 10, fontSize: 13 },
  state: { fontSize: 12, marginTop: 6 },
  btn: { ...primaryBtn, marginTop: 4 },
  btnText: { color: '#fff', fontWeight: '700', fontSize: 16 },
  status: { color: colors.muted, marginTop: 12, fontSize: 13 },
});

const c = StyleSheet.create({
  wrap: { flex: 1, padding: 12 },
  bar: { flexDirection: 'row', alignItems: 'center', marginBottom: 10 },
  name: {
    flex: 1,
    color: colors.text,
    borderColor: colors.border,
    borderWidth: 1,
    borderRadius: 8,
    paddingHorizontal: 10,
    paddingVertical: 8,
  },
  run: { marginLeft: 8, borderRadius: 8, paddingVertical: 10, paddingHorizontal: 16 },
  runText: { color: '#fff', fontWeight: '700' },
  editor: {
    ...box,
    flex: 1,
    color: colors.text,
    padding: 10,
    fontFamily: mono,
    fontSize: 13,
    textAlign: 'left',
  },
  preview: { flex: 1, borderRadius: 10, overflow: 'hidden', backgroundColor: '#fff' },
  actions: { flexDirection: 'row', flexWrap: 'wrap', marginTop: 10 },
  act: {
    backgroundColor: colors.card,
    borderColor: colors.border,
    borderWidth: 1,
    borderRadius: 8,
    paddingVertical: 8,
    paddingHorizontal: 12,
    marginRight: 8,
    marginBottom: 8,
  },
  actText: { color: colors.text, fontSize: 13 },
  status: { color: colors.muted, fontSize: 13 },
  list: { ...box, position: 'absolute', left: 12, right: 12, bottom: 70, maxHeight: 260, padding: 8 },
  item: { padding: 12, borderBottomColor: colors.border, borderBottomWidth: 1 },
  itemText: { color: colors.text, fontSize: 15 },
  close: { color: colors.primary, textAlign: 'center', paddingTop: 10 },
});

const e = StyleSheet.create({
  wrap: { padding: 16, paddingBottom: 40 },
  title: { color: colors.text, fontSize: 22, fontWeight: '700', marginBottom: 6 },
  hint: { color: colors.muted, fontSize: 13, marginBottom: 16 },
  engines: { flexDirection: 'row', marginBottom: 12 },
  engine: { ...box, flex: 1, padding: 12, alignItems: 'center', marginHorizontal: 4 },
  engineOn: { backgroundColor: colors.primary, borderColor: colors.primary },
  engineText: { color: colors.text, fontWeight: '700', fontSize: 16 },
  keyState: { color: colors.muted, fontSize: 11, marginTop: 4 },
  input: { ...box, minHeight: 120, color: colors.text, padding: 10, fontSize: 15 },
  btn: { ...primaryBtn, marginTop: 12 },
  btnText: { color: '#fff', fontWeight: '700', fontSize: 16 },
  error: { color: colors.danger, marginTop: 12, fontSize: 13 },
  doneBox: { ...box, marginTop: 12, borderColor: colors.success, padding: 12 },
  done: { color: colors.success, fontSize: 14 },
  doneRow: { flexDirection: 'row', justifyContent: 'space-between', marginTop: 10 },
  link: { color: colors.primary, fontSize: 14 },
});


name: Build App APK

on:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Create Expo project and add the app code
        run: |
          npx --yes create-expo-app@latest build --template blank --no-install
          cd build
          npm install
          npx expo install expo-secure-store expo-file-system expo-sharing react-native-webview @react-native-async-storage/async-storage
          rm -f App.js
          cp ../App.js .

      - name: Set app name and Android package
        run: |
          cd build
          node -e "const fs=require('fs');const j=JSON.parse(fs.readFileSync('app.json','utf8'));j.expo.name='AI Builder';j.expo.android=Object.assign({},j.expo.android,{package:'com.aibuilder.mobile'});fs.writeFileSync('app.json',JSON.stringify(j,null,2))"

      - name: Generate Android project
        run: |
          cd build
          npx expo prebuild --platform android --no-install

      - name: Build release APK
        run: |
          cd build/android
          chmod +x gradlew
          ./gradlew assembleRelease

      - name: Publish APK as release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          cp build/android/app/build/outputs/apk/release/app-release.apk app.apk
          gh release create "app-${{ github.run_number }}" app.apk \
            --title "AI Builder ${{ github.run_number }}" \
            --notes "APK of the AI Builder app"


        create table if not exists public.projects (
  name text primary key,
  code text not null default '',
  updated_at timestamptz not null default now()
);

alter table public.projects enable row level security;

drop policy if exists "personal use" on public.projects;
create policy "personal use" on public.projects
  for all to anon
  using (true)
  with check (true);

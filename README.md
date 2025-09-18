/******** 設定 ********/
const CFG = {
  // あなたが指定したID群
  INBOX_FOLDER_ID: '1mfu-evxXOqEjRWvHnBC0Gfn358QgOH3Q',   // INBOX
  MEMO_FOLDER_ID:  '1h3jk6r1T4LmRyuMJ5GQUJAMzuLs18kyU',   // 元画像の保存先
  PDF_FOLDER_ID:   '1oB9M4qv-VTI5LH2RIgjLB-_ZUexdcl3b',   // PDF保存先
  SS_ID:           '1PageTkBZ500Od9_412uC_G6F1xTaKFtsDFzyPcOiVrk', // 請求書テンプレを含むスプシ
  TEMPLATE_SHEET:  '請求書テンプレ',                        // テンプレのタブ名

  // 必ずあなたのアドレスに変更
  SELF_EMAIL:      'yusakuf1.nsd@gmail.com',

  MODEL:           'gemini-1.5-flash',
  TIMEZONE:        'Asia/Tokyo',
  MAX_ROWS:        10,        // 明細行数 B20〜B29
  KEYWORD:         '請求書'   // ファイル名に含まれるキーワード
};

/******** SA(JWT)→アクセストークン（Gemini呼び出し用） ********/
function getAccessToken_() {
  const p = PropertiesService.getScriptProperties();
  let privateKey = p.getProperty('GCP_SA_PRIVATE_KEY');
  if (!privateKey) throw new Error('GCP_SA_PRIVATE_KEY が未設定です');
  privateKey = privateKey.replace(/\\n/g, '\n');

  const header = Utilities.base64EncodeWebSafe(JSON.stringify({alg:'RS256',typ:'JWT'}));
  const now = Math.floor(Date.now()/1000);
  const payload = Utilities.base64EncodeWebSafe(JSON.stringify({
  iss: p.getProperty('GCP_SA_EMAIL'),
  scope: 'https://www.googleapis.com/auth/generative-language', // ★ここを修正
  aud: 'https://oauth2.googleapis.com/token',
  exp: now + 3600, iat: now
}));

  const toSign = `${header}.${payload}`;
  const sig = Utilities.base64EncodeWebSafe(Utilities.computeRsaSha256Signature(toSign, privateKey));
  const jwt = `${toSign}.${sig}`;

  const res = UrlFetchApp.fetch('https://oauth2.googleapis.com/token', {
    method:'post',
    payload:{grant_type:'urn:ietf:params:oauth:grant-type:jwt-bearer', assertion: jwt}
  });
  const json = JSON.parse(res.getContentText());
  if (!json.access_token) throw new Error('アクセストークン取得失敗: ' + res.getContentText());
  return json.access_token;
}

/******** Geminiで画像→構造化抽出(JSON) ********/
function extractByGemini_(file, opt__retried) {
  const token = getAccessToken_();
  const blob = file.getBlob();

  const prompt = `
あなたは経理アシスタントです。画像（請求書メモ）から下記のJSONのみを返してください。説明文やコードブロックは禁止。

{
  "date": "YYYY-MM-DD",
  "client": "文字列",
  "items": [
    {"category":"材料費|運搬費|手数料","name":"文字列","qty": 数値,"unit_price": 数値}
  ]
}
...省略...
`;

  const body = {
  contents: [{
    role: "user",
    parts: [
      {text: prompt},
      {inline_data: {mime_type: blob.getContentType(), data: Utilities.base64Encode(blob.getBytes())}}
    ]
  }],
  generationConfig: {
    temperature: 0,
    topK: 1,
    topP: 0,
    maxOutputTokens: 1024
    // responseMimeType は削除
  }
};


  const url = `https://generativelanguage.googleapis.com/v1/models/${CFG.MODEL}:generateContent`;
  const res = UrlFetchApp.fetch(url, {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: `Bearer ${token}` },
    payload: JSON.stringify(body),
    muteHttpExceptions: true
  });

  // ★ここが新しい！必ず生レスポンスをログ出力
  const status = res.getResponseCode();
  const raw = res.getContentText();
  Logger.log('[Gemini] status=' + status);
  Logger.log('[Gemini] raw=' + raw);

  let out;
  try { out = JSON.parse(raw); } catch(_) { out = null; }
  if (!out) {
    if (!opt__retried) return extractByGemini_(file, true);
    return null;
  }
  if (out.error) {
    Logger.log('[Gemini] error=' + JSON.stringify(out.error));
    if (!opt__retried) return extractByGemini_(file, true);
    return null;
  }

  const text = out?.candidates?.[0]?.content?.parts?.map(p => p.text).join('') || '';
  if (!text) {
    Logger.log('[Gemini] candidatesなし');
    if (!opt__retried) return extractByGemini_(file, true);
    return null;
  }

  try {
  // ★ まずコードブロックを除去
  let clean = text.trim();
  clean = clean.replace(/^```json\s*/i, '').replace(/^```/, '').replace(/```$/,'').trim();

  const parsed = JSON.parse(clean);

 // 正規化処理（カテゴリの自動補正を仕様通りに）
parsed.items = Array.isArray(parsed.items) ? parsed.items : [];

function normalizeCategory(cat, name){
  const t = String(cat || '') + String(name || '');
  // 優先順：産業廃棄物処理費→人件費→諸経費→材料費→運搬費→手数料
  if (/(産業?廃棄物処理費?|産)/.test(t)) return '産業廃棄物処理費';
  if (/(人件費?|人)/.test(t))             return '人件費';
  if (/(諸経費?|諸)/.test(t))             return '諸経費';
  if (/(材料|材)/.test(t))                return '材料費';
  if (/運/.test(t))                       return '運搬費';
  if (/手/.test(t))                       return '手数料';

  // 既に正しい表記ならそのまま
  const known = ['産業廃棄物処理費','人件費','諸経費','材料費','運搬費','手数料'];
  if (known.includes(String(cat))) return String(cat);

  // いずれにも当てはまらなければデフォルト
  return '手数料';
}

parsed.items = parsed.items.map(it => {
  const name = it.name || it.category || '不明';
  const category = normalizeCategory(it.category, name);
  const qty = Number(String(it.qty ?? 1).replace(/[^\d.-]/g,'')) || 1;
  const unit_price = Number(String(it.unit_price ?? 0).replace(/[^\d.-]/g,'')) || 0;
  return { category, name, qty, unit_price };
});

// ★ 発行日は必ず「作成日（今日）」に統一して年ズレを防止
parsed.date = Utilities.formatDate(new Date(), CFG.TIMEZONE, 'yyyy-MM-dd');
parsed.client = parsed.client || '不明';
return parsed;




} catch(e) {
  Logger.log('Gemini応答のJSON化失敗(再度パース不可): ' + text);
  if (!opt__retried) return extractByGemini_(file, true);
  return null;
}

}


/******** テンプレを複製し、指定セルに書き込む ********/
function makeInvoiceFromPayload_(payload, createdDateStr) {
  const ss = SpreadsheetApp.openById(CFG.SS_ID);
  const src = ss.getSheetByName(CFG.TEMPLATE_SHEET);
  if (!src) throw new Error('テンプレートタブが見つかりません: ' + CFG.TEMPLATE_SHEET);

  const tz = CFG.TIMEZONE;
  const invoiceId = 'INV-' + Utilities.formatDate(new Date(), tz, 'yyyyMMdd-HHmmss');

  // 呼び出し側で作成日を決定済みだが、念のためフォールバックを用意
  const issuedDateStr = createdDateStr || Utilities.formatDate(new Date(), tz, 'yyyy-MM-dd');

  const safeClient = String(payload.client).replace(/[\\/:*?"<>|]/g,'');
  // ★ タブ名 = 請求書作成日_請求先（作成日はハイフン抜き）
  const tabName = `${issuedDateStr.replace(/-/g,'')}_${safeClient}`.slice(0, 99);

  const sh = src.copyTo(ss).setName(tabName);

  // ★ 発行日（D2）は作成日で固定
  sh.getRange('D2').setValue(issuedDateStr);
  sh.getRange('A6').setValue(payload.client); // 請求先

  const startRow = 20;
  const maxRows = CFG.MAX_ROWS;
  sh.getRange(startRow, 2, maxRows, 3).clearContent();

  payload.items.slice(0, maxRows).forEach((it, i) => {
    const r = startRow + i;
    sh.getRange(`B${r}`).setValue(it.name);
    sh.getRange(`C${r}`).setValue(it.unit_price);
    sh.getRange(`D${r}`).setValue(it.qty);
  });

  sh.getRange('C15').setFormula('=E32');

  return {invoiceId, sheetName: tabName};
}

/******** PDF出力 ********/
function exportSheetPdf_(spreadsheetId, sheetId, filename) {
  const url = `https://docs.google.com/spreadsheets/d/${spreadsheetId}/export?` +
    `format=pdf&gid=${sheetId}&size=a4&portrait=true&fitw=true&gridlines=false&sheetnames=false&printtitle=false&fzr=true`;
  const token = ScriptApp.getOAuthToken();
  return UrlFetchApp.fetch(url, {headers:{Authorization:`Bearer ${token}`}}).getBlob().setName(filename);
}

/******** メイン ********/
function run(force=false){
  const inbox = DriveApp.getFolderById(CFG.INBOX_FOLDER_ID);
  const files = inbox.getFiles();
  let any = false;

  while (files.hasNext()) {
    any = true;
    const f = files.next();
    const name = f.getName();
    const desc = f.getDescription() || '';
    const mime = f.getMimeType();
    Logger.log('検査: name="%s" mime="%s" desc="%s"', name, mime, desc);

    if (!force && (desc||'').includes('PROCESSED')) {
      Logger.log('→ スキップ: 既にPROCESSED');
      continue;
    }

    if (!/^(image|application\/pdf)/i.test(mime)) {
      Logger.log('→ スキップ: 画像/PDFではない');
      continue;
    }

    const base = (name + ' ' + desc).replace(/\s+/g,'');
    if (!base.includes(CFG.KEYWORD)) {
      Logger.log('→ スキップ: キーワード不一致');
      continue;
    }

    try {
      Logger.log('→ Gemini抽出を開始');
      const payload = extractByGemini_(f);
      Logger.log('→ Gemini結果: ' + JSON.stringify(payload));

      if (!payload || !payload.items || payload.items.length === 0) {
        Logger.log('→ 中断: 明細抽出できず');
        f.setDescription('PROCESSED:NOMATCH');
        continue;
      }

      Logger.log('→ メモフォルダへ移動');
      DriveApp.getFolderById(CFG.MEMO_FOLDER_ID).addFile(f);
      inbox.removeFile(f);

      Logger.log('→ 請求書タブを作成');
// ★ 作成日をここで確定させ、以降すべてこの日付で統一
const createdDateStr = Utilities.formatDate(new Date(), CFG.TIMEZONE, 'yyyy-MM-dd');
const {invoiceId, sheetName} = makeInvoiceFromPayload_(payload, createdDateStr);
Logger.log('→ invoiceId=' + invoiceId + ' sheetName=' + sheetName);

// ★ テンプレではなく “新規作成タブ(sheetName)” をPDF化して送る
Logger.log('→ PDF化して保存/送信');
const ss = SpreadsheetApp.openById(CFG.SS_ID);
const invSheet = ss.getSheetByName(sheetName);
const pdfBlob = exportSheetPdf_(CFG.SS_ID, invSheet.getSheetId(), `請求書_${sheetName}.pdf`);
DriveApp.getFolderById(CFG.PDF_FOLDER_ID).createFile(pdfBlob);

GmailApp.sendEmail(
  CFG.SELF_EMAIL,
  `請求書発行: ${sheetName}`,
  `請求書を発行しました。\n請求先: ${payload.client}\n発行日: ${createdDateStr}\nタブ名: ${sheetName}`,
  {attachments:[pdfBlob]}
);




      f.setDescription(`PROCESSED:OK:${invoiceId}`);
      Logger.log('→ 完了: ' + name);

    } catch (e) {
      Logger.log('→ 例外: ' + (e && e.stack || e));
      try { f.setDescription('PROCESSED:ERROR:' + e); } catch (_) {}
    }
  }

  if (!any) Logger.log('INBOXにファイルがありません');
}

/******** ユーティリティ ********/
function diag_inboxCheck(){
  const folder = DriveApp.getFolderById(CFG.INBOX_FOLDER_ID);
  Logger.log("INBOXフォルダ名: " + folder.getName());
  const files = folder.getFiles();
  let count=0;
  while (files.hasNext()){
    const f=files.next();
    Logger.log('  - ' + f.getName() + ' (' + f.getMimeType() + ') desc=' + f.getDescription());
    count++;
  }
  if (!count) Logger.log("INBOXにファイルなし");
}

function resetProcessed(namePart){
  const folder = DriveApp.getFolderById(CFG.INBOX_FOLDER_ID);
  const it = folder.getFiles();
  let n=0;
  while(it.hasNext()){
    const f=it.next();
    if (!namePart || f.getName().includes(namePart)){
      f.setDescription(''); n++;
    }
  }
  Logger.log('説明クリア: ' + n + ' 件');
}

function checkProps(){
  const p = PropertiesService.getScriptProperties();
  Logger.log("EMAIL: " + p.getProperty('GCP_SA_EMAIL'));
  Logger.log("KEY(先頭40): " + (p.getProperty('GCP_SA_PRIVATE_KEY') || '').substring(0,40));
}

function testToken(){
  const token = getAccessToken_();
  Logger.log("Access Token: " + token);
}

function setupTimeTrigger(){
  ScriptApp.newTrigger('run').timeBased().everyMinutes(5).create();
}
function run_force(){
  run(true); // PROCESSED無視モード
}

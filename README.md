// Variables used by Scriptable.
// These must be at the very top of the file. Do not edit.
// icon-color: orange; icon-glyph: utensils;

/**
 * ============================================================================
 * 🥢 聚餐分帳計算機 (iPhone iOS Scriptable 專用版)
 * ============================================================================
 * 功能特色：
 * 1. 專為 iPhone 設計的 iOS 原生感介面（支援底部導覽列、安全區域、純離線執行）
 * 2. 支援「個人獨享」、「全員平分」、「指定成員合點」三種分攤模式
 * 3. 支援服務費 (0%/5%/10%/自訂)、優惠折扣、額外費用與找零取整設定
 * 4. 自動透過 Scriptable FileManager 將資料保存在 iPhone 本機，重開不遺失
 * 5. 整合 iOS 原生剪貼簿 (Pasteboard) 與系統分享選單 (ShareSheet)，一鍵傳 LINE
 * 6. 支援 iOS 桌面小工具 (Widget)：將此腳本加入桌面即可隨時查看每人應付金額
 * ============================================================================
 */

const FILE_NAME = "meal_split_state_v1.json";
const fm = FileManager.local();
const dataPath = fm.joinPath(fm.documentsDirectory(), FILE_NAME);

// 預設示範資料
const DEFAULT_STATE = {
  members: ["小明", "小美", "阿豪", "雅婷"],
  items: [
    { id: "item_1", name: "頂級厚切牛排", price: 420, qty: 1, mode: "individual", target: "小明" },
    { id: "item_2", name: "松露野菇義大利麵", price: 320, qty: 1, mode: "individual", target: "小美" },
    { id: "item_3", name: "香煎鮭魚燉飯", price: 350, qty: 1, mode: "individual", target: "雅婷" },
    { id: "item_4", name: "美式雙層起司堡", price: 280, qty: 1, mode: "individual", target: "阿豪" },
    { id: "item_5", name: "綜合海鮮大拼盤 (公家)", price: 480, qty: 1, mode: "all", target: "all" },
    { id: "item_6", name: "金牌精釀生啤酒 (2人合點)", price: 160, qty: 2, mode: "custom", target: ["小明", "阿豪"] }
  ],
  serviceRate: 10,
  discount: 100,
  otherFee: 0,
  feeSplitRule: "proportional",
  rounding: "none",
  paymentNote: "街口支付帳號: 901234567 或 台新 (812) 2888-12345678"
};

// 讀取與儲存本機狀態
function loadStateFromDisk() {
  try {
    if (fm.fileExists(dataPath)) {
      const raw = fm.readString(dataPath);
      const parsed = JSON.parse(raw);
      if (parsed && Array.isArray(parsed.members) && Array.isArray(parsed.items)) {
        return Object.assign({}, DEFAULT_STATE, parsed);
      }
    }
  } catch (e) {
    console.error("讀取存檔失敗: " + e);
  }
  return JSON.parse(JSON.stringify(DEFAULT_STATE));
}

function saveStateToDisk(newState) {
  try {
    fm.writeString(dataPath, JSON.stringify(newState, null, 2));
  } catch (e) {
    console.error("寫入存檔失敗: " + e);
  }
}

// 供 iOS 桌面 Widget 使用的試算函式
function calculateSummaryForWidget(state) {
  const memberCount = state.members.length;
  const stats = {};
  state.members.forEach(m => {
    stats[m] = { indTotal: 0, sharedTotal: 0, feeShare: 0, roundedPay: 0 };
  });

  let itemsTotal = 0;
  state.items.forEach(item => {
    const total = (Number(item.price) || 0) * (Number(item.qty) || 1);
    itemsTotal += total;
    if (item.mode === "individual" && stats[item.target]) {
      stats[item.target].indTotal += total;
    } else if (item.mode === "all" && memberCount > 0) {
      const per = total / memberCount;
      state.members.forEach(m => (stats[m].sharedTotal += per));
    } else if (item.mode === "custom" && Array.isArray(item.target)) {
      const valid = item.target.filter(m => state.members.includes(m));
      if (valid.length > 0) {
        const per = total / valid.length;
        valid.forEach(m => (stats[m].sharedTotal += per));
      }
    }
  });

  const serviceFee = (itemsTotal * (Number(state.serviceRate) || 0)) / 100;
  const netAdj = serviceFee + (Number(state.otherFee) || 0) - (Number(state.discount) || 0);
  const grandTotal = Math.max(0, itemsTotal + netAdj);

  state.members.forEach(m => {
    const sub = stats[m].indTotal + stats[m].sharedTotal;
    if (state.feeSplitRule === "proportional" && itemsTotal > 0) {
      stats[m].feeShare = netAdj * (sub / itemsTotal);
    } else {
      stats[m].feeShare = memberCount > 0 ? netAdj / memberCount : 0;
    }
    const rawPay = Math.max(0, sub + stats[m].feeShare);
    if (state.rounding === "round5") {
      stats[m].roundedPay = Math.ceil(rawPay / 5) * 5;
    } else if (state.rounding === "round10") {
      stats[m].roundedPay = Math.ceil(rawPay / 10) * 10;
    } else if (state.rounding === "exact") {
      stats[m].roundedPay = Math.round(rawPay * 10) / 10;
    } else {
      stats[m].roundedPay = Math.round(rawPay);
    }
  });

  return { grandTotal: Math.round(grandTotal), stats };
}

// 建立 iOS 桌面小工具 (Widget)
function buildWidget(state) {
  const widget = new ListWidget();
  const gradient = new LinearGradient();
  gradient.colors = [new Color("#1C1C1E"), new Color("#2C2C2E")];
  gradient.locations = [0, 1];
  widget.backgroundGradient = gradient;
  widget.setPadding(14, 16, 14, 16);

  const res = calculateSummaryForWidget(state);

  const header = widget.addStack();
  header.layoutHorizontally();
  header.centerAlignContent();

  const title = header.addText("🥢 聚餐分帳");
  title.font = Font.boldSystemFont(13);
  title.textColor = new Color("#FF9F0A");

  header.addSpacer();

  const totalTxt = header.addText(`總計 $${res.grandTotal.toLocaleString()}`);
  totalTxt.font = Font.boldSystemFont(12);
  totalTxt.textColor = Color.white();

  widget.addSpacer(8);

  if (state.members.length === 0) {
    const empty = widget.addText("點擊進入新增成員與餐點");
    empty.font = Font.systemFont(12);
    empty.textColor = new Color("#8E8E93");
  } else {
    const maxShow = Math.min(state.members.length, 4);
    for (let i = 0; i < maxShow; i++) {
      const m = state.members[i];
      const row = widget.addStack();
      row.layoutHorizontally();
      row.centerAlignContent();

      const nameTxt = row.addText(m);
      nameTxt.font = Font.mediumSystemFont(13);
      nameTxt.textColor = new Color("#E5E5EA");

      row.addSpacer();

      const payTxt = row.addText(`NT$ ${res.stats[m].roundedPay.toLocaleString()}`);
      payTxt.font = Font.boldSystemFont(13);
      payTxt.textColor = new Color("#30D158");

      if (i < maxShow - 1) widget.addSpacer(4);
    }
    if (state.members.length > 4) {
      widget.addSpacer(4);
      const more = widget.addText(`...及其他 ${state.members.length - 4} 人`);
      more.font = Font.systemFont(10);
      more.textColor = new Color("#8E8E93");
    }
  }

  return widget;
}

// 產生內嵌於 Scriptable WebView 的 iPhone iOS 介面 HTML (100% 離線無外部依賴)
function buildIPhoneHTML(initialState) {
  const stateJson = JSON.stringify(initialState);
  return `<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
<title>聚餐分帳</title>
<style>
  :root {
    --ios-bg: #F2F2F7;
    --ios-card: #FFFFFF;
    --ios-orange: #FF9500;
    --ios-orange-dark: #E08600;
    --ios-green: #34C759;
    --ios-blue: #007AFF;
    --ios-red: #FF3B30;
    --ios-text: #1C1C1E;
    --ios-sub: #8E8E93;
    --ios-sep: #E5E5EA;
  }
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  body {
    margin: 0; padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "Helvetica Neue", sans-serif;
    background: var(--ios-bg);
    color: var(--ios-text);
    padding-bottom: calc(88px + env(safe-area-inset-bottom));
    user-select: none;
  }
  input, select, button { font-family: inherit; }

  /* 頂部 iOS 導覽列 */
  .nav-bar {
    position: sticky; top: 0; z-index: 30;
    background: rgba(242, 242, 247, 0.88);
    backdrop-filter: blur(20px); -webkit-backdrop-filter: blur(20px);
    border-bottom: 0.5px solid var(--ios-sep);
    padding: 10px 16px;
  }
  .nav-top {
    display: flex; align-items: center; justify-content: space-between;
  }
  .nav-title {
    font-size: 20px; font-weight: 800; letter-spacing: -0.4px;
    display: flex; align-items: center; gap: 6px;
  }
  .nav-actions { display: flex; gap: 8px; }
  .pill-btn {
    border: none; border-radius: 999px;
    padding: 6px 12px; font-size: 12px; font-weight: 600;
    cursor: pointer; display: inline-flex; align-items: center; gap: 4px;
  }
  .pill-orange { background: rgba(255, 149, 0, 0.15); color: #C96E00; }
  .pill-red { background: rgba(255, 59, 48, 0.12); color: var(--ios-red); }
  .pill-green { background: var(--ios-green); color: #fff; }

  /* 即時總額摘要橫幅 */
  .summary-strip {
    margin: 10px 16px 0;
    background: linear-gradient(135deg, #FF9500, #FF5E3A);
    color: #fff; border-radius: 16px; padding: 14px 16px;
    display: flex; justify-content: space-between; align-items: center;
    box-shadow: 0 6px 16px rgba(255, 149, 0, 0.25);
  }
  .summary-strip .label { font-size: 12px; opacity: 0.9; font-weight: 500; }
  .summary-strip .total { font-size: 26px; font-weight: 800; letter-spacing: -0.5px; margin-top: 2px; }
  .summary-strip .meta { text-align: right; font-size: 11px; opacity: 0.92; line-height: 1.5; }

  /* 內容區塊 */
  .tab-pane { display: none; padding: 14px 16px; }
  .tab-pane.active { display: block; }

  .section-header {
    font-size: 13px; font-weight: 600; color: var(--ios-sub);
    text-transform: uppercase; margin: 6px 4px 8px;
    display: flex; justify-content: space-between; align-items: center;
  }
  .ios-card {
    background: var(--ios-card); border-radius: 14px;
    padding: 14px; margin-bottom: 16px;
    box-shadow: 0 1px 2px rgba(0,0,0,0.03);
  }

  /* 表單元件 */
  .input-row { display: flex; gap: 8px; }
  .ios-input {
    width: 100%; padding: 11px 12px;
    font-size: 15px; border: 1px solid var(--ios-sep);
    border-radius: 10px; background: #FAFAFC; color: var(--ios-text);
    outline: none; -webkit-appearance: none;
  }
  .ios-input:focus { border-color: var(--ios-orange); background: #fff; }
  .btn-primary {
    background: var(--ios-orange); color: #fff; border: none;
    border-radius: 10px; padding: 11px 16px; font-size: 15px; font-weight: 600;
    white-space: nowrap; cursor: pointer;
  }
  .btn-primary:active { opacity: 0.8; }

  /* 成員膠囊 */
  .preset-bar { display: flex; gap: 6px; flex-wrap: wrap; margin-top: 10px; align-items: center; }
  .preset-chip {
    background: var(--ios-bg); border: none; border-radius: 8px;
    padding: 5px 10px; font-size: 12px; color: #3A3A3C; font-weight: 500;
  }
  .member-chips { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 12px; }
  .member-chip {
    display: inline-flex; align-items: center; gap: 6px;
    background: #FFF8F0; border: 1px solid #FFE0B2;
    padding: 6px 10px; border-radius: 999px; font-size: 14px; font-weight: 600;
  }
  .member-chip .avatar {
    width: 22px; height: 22px; border-radius: 50%;
    background: var(--ios-orange); color: #fff; font-size: 12px;
    display: flex; align-items: center; justify-content: center;
  }
  .member-chip .del {
    border: none; background: none; color: var(--ios-sub);
    font-size: 16px; line-height: 1; padding: 0 2px;
  }

  /* 分段控制器 (Segmented Control) */
  .segmented {
    display: grid; grid-template-columns: repeat(3, 1fr);
    background: var(--ios-bg); padding: 3px; border-radius: 10px; margin: 8px 0 12px;
  }
  .seg-btn {
    border: none; background: transparent; padding: 7px 4px;
    font-size: 13px; font-weight: 600; color: var(--ios-sub);
    border-radius: 8px; transition: 0.15s;
  }
  .seg-btn.active {
    background: #fff; color: var(--ios-text);
    box-shadow: 0 1px 4px rgba(0,0,0,0.1);
  }

  /* 自訂成員勾選 */
  .check-grid { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 8px; }
  .check-pill {
    display: inline-flex; align-items: center; gap: 6px;
    padding: 6px 12px; border-radius: 999px; font-size: 13px; font-weight: 600;
    background: var(--ios-bg); color: var(--ios-text); border: 1.5px solid transparent;
  }
  .check-pill.selected {
    background: rgba(255, 149, 0, 0.14); color: #C96E00; border-color: var(--ios-orange);
  }

  /* 餐點清單 */
  .item-row {
    display: flex; justify-content: space-between; align-items: center;
    padding: 11px 0; border-bottom: 0.5px solid var(--ios-sep);
  }
  .item-row:last-child { border-bottom: none; }
  .item-title { font-size: 15px; font-weight: 600; }
  .item-sub { font-size: 12px; color: var(--ios-sub); margin-top: 3px; }
  .item-right { display: flex; align-items: center; gap: 10px; }
  .item-price { font-size: 15px; font-weight: 700; }
  .btn-del-item {
    border: none; background: rgba(255, 59, 48, 0.1); color: var(--ios-red);
    width: 28px; height: 28px; border-radius: 50%; font-size: 14px; font-weight: bold;
  }

  /* 成員應付卡片 */
  .person-card {
    background: #fff; border-radius: 16px; padding: 14px; margin-bottom: 12px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.04);
  }
  .person-top {
    display: flex; justify-content: space-between; align-items: center;
    border-bottom: 0.5px solid var(--ios-sep); padding-bottom: 10px;
  }
  .person-name { font-size: 17px; font-weight: 700; display: flex; align-items: center; gap: 8px; }
  .person-amount { font-size: 20px; font-weight: 800; color: var(--ios-orange); }
  .person-details { font-size: 12px; color: #555; padding-top: 10px; line-height: 1.6; }
  .person-actions { margin-top: 10px; display: flex; justify-content: flex-end; gap: 8px; }

  /* 底部 iOS Tab Bar */
  .tab-bar {
    position: fixed; bottom: 0; left: 0; right: 0; z-index: 40;
    background: rgba(255, 255, 255, 0.92);
    backdrop-filter: blur(20px); -webkit-backdrop-filter: blur(20px);
    border-top: 0.5px solid var(--ios-sep);
    display: grid; grid-template-columns: repeat(4, 1fr);
    padding-top: 6px;
    padding-bottom: calc(6px + env(safe-area-inset-bottom));
  }
  .tab-btn {
    border: none; background: none; display: flex; flex-direction: column;
    align-items: center; gap: 2px; color: var(--ios-sub);
    font-size: 10px; font-weight: 600; padding: 4px 0;
  }
  .tab-btn .icon { font-size: 20px; line-height: 1.1; }
  .tab-btn.active { color: var(--ios-orange); }

  /* Toast */
  #toast {
    position: fixed; top: 18px; left: 50%; transform: translateX(-50%) translateY(-80px);
    background: rgba(28, 28, 30, 0.95); color: #fff;
    padding: 10px 18px; border-radius: 999px; font-size: 13px; font-weight: 600;
    z-index: 100; transition: 0.25s cubic-bezier(0.2, 0.8, 0.2, 1);
    box-shadow: 0 6px 20px rgba(0,0,0,0.2); pointer-events: none;
  }
  #toast.show { transform: translateX(-50%) translateY(0); }
</style>
</head>
<body>

  <div id="toast">已複製分帳明細！</div>

  <!-- 頂部導覽列 -->
  <div class="nav-bar">
    <div class="nav-top">
      <div class="nav-title">🥢 聚餐分帳</div>
      <div class="nav-actions">
        <button type="button" class="pill-btn pill-orange" id="btn-sample">範例</button>
        <button type="button" class="pill-btn pill-red" id="btn-reset">清空</button>
        <button type="button" class="pill-btn pill-green" id="btn-share-top">分享帳單</button>
      </div>
    </div>
  </div>

  <!-- 即時總覽橫幅 -->
  <div class="summary-strip">
    <div>
      <div class="label">帳單實付總額</div>
      <div class="total" id="top-grand-total">NT$ 0</div>
    </div>
    <div class="meta">
      <div id="top-meta-members">👥 0 人用餐</div>
      <div id="top-meta-items">🍽️ 0 項餐點</div>
      <div id="top-meta-fee">🧾 服務費 0%</div>
    </div>
  </div>

  <!-- 分頁 1：成員與點餐 -->
  <div class="tab-pane active" id="tab-order">
    <div class="section-header">
      <span>1. 用餐成員名單</span>
      <span id="member-count-label">0 人</span>
    </div>
    <div class="ios-card">
      <div class="input-row">
        <input type="text" id="member-input" class="ios-input" placeholder="輸入朋友名字 (如：小明)">
        <button type="button" class="btn-primary" id="btn-add-member">＋新增</button>
      </div>
      <div class="preset-bar">
        <span style="font-size:11px;color:var(--ios-sub);">快選：</span>
        <button type="button" class="preset-chip" data-preset="小明">+小明</button>
        <button type="button" class="preset-chip" data-preset="小美">+小美</button>
        <button type="button" class="preset-chip" data-preset="阿豪">+阿豪</button>
        <button type="button" class="preset-chip" data-preset="雅婷">+雅婷</button>
      </div>
      <div class="member-chips" id="member-chips"></div>
    </div>

    <div class="section-header">
      <span>2. 新增餐點品項</span>
    </div>
    <div class="ios-card">
      <div style="margin-bottom:10px;">
        <input type="text" id="item-name" class="ios-input" placeholder="餐點名稱 (如：厚切牛排、海鮮拼盤)">
      </div>
      <div class="input-row" style="margin-bottom:10px;">
        <input type="number" id="item-price" class="ios-input" placeholder="單價 NT$" inputmode="decimal">
        <input type="number" id="item-qty" class="ios-input" style="max-width:95px;text-align:center;" value="1" min="1" inputmode="numeric">
      </div>

      <div style="font-size:12px;color:var(--ios-sub);font-weight:600;">分攤方式</div>
      <div class="segmented">
        <button type="button" class="seg-btn active" data-mode="individual">👤 個人獨享</button>
        <button type="button" class="seg-btn" data-mode="all">👥 全員平分</button>
        <button type="button" class="seg-btn" data-mode="custom">🎯 指定分攤</button>
      </div>

      <div id="box-individual" style="margin-bottom:12px;">
        <select id="select-member" class="ios-input"></select>
      </div>

      <div id="box-custom" style="display:none;margin-bottom:12px;">
        <div style="font-size:12px;color:var(--ios-sub);">點選要一起分攤此餐點的人：</div>
        <div class="check-grid" id="custom-checks"></div>
      </div>

      <button type="button" class="btn-primary" id="btn-add-item" style="width:100%;">＋ 加入餐點清單</button>
    </div>
  </div>

  <!-- 分頁 2：已點清單 -->
  <div class="tab-pane" id="tab-list">
    <div class="section-header">
      <span>已點餐點明細</span>
      <span id="list-subtotal">小計 NT$ 0</span>
    </div>
    <div class="ios-card" id="items-container"></div>
  </div>

  <!-- 分頁 3：服務費與折扣設定 -->
  <div class="tab-pane" id="tab-settings">
    <div class="section-header"><span>服務費與附加調整</span></div>
    <div class="ios-card">
      <div style="font-size:13px;font-weight:600;margin-bottom:6px;">服務費比例 (%)</div>
      <div class="segmented" style="grid-template-columns:repeat(4,1fr);">
        <button type="button" class="seg-btn fee-btn" data-rate="0">0%</button>
        <button type="button" class="seg-btn fee-btn" data-rate="5">5%</button>
        <button type="button" class="seg-btn fee-btn active" data-rate="10">10%</button>
        <button type="button" class="seg-btn fee-btn" data-rate="custom">自訂</button>
      </div>
      <input type="number" id="input-service-rate" class="ios-input" placeholder="自訂服務費 %" style="margin-bottom:12px;" value="10">

      <div style="font-size:13px;font-weight:600;margin-bottom:6px;">折扣折抵金額 (NT$)</div>
      <input type="number" id="input-discount" class="ios-input" placeholder="0" style="margin-bottom:12px;" inputmode="decimal">

      <div style="font-size:13px;font-weight:600;margin-bottom:6px;">其他雜費 (如開瓶費/包廂費 NT$)</div>
      <input type="number" id="input-other-fee" class="ios-input" placeholder="0" style="margin-bottom:12px;" inputmode="decimal">

      <div style="font-size:13px;font-weight:600;margin-bottom:6px;">附加費分攤規則</div>
      <select id="select-fee-rule" class="ios-input" style="margin-bottom:12px;">
        <option value="proportional">依個人消費金額比例分攤 (推薦)</option>
        <option value="equal">全體成員人頭平分</option>
      </select>

      <div style="font-size:13px;font-weight:600;margin-bottom:6px;">零錢取整方式</div>
      <select id="select-rounding" class="ios-input" style="margin-bottom:12px;">
        <option value="none">四捨五入到 1 元 (預設)</option>
        <option value="round5">無條件進位至 5 元整數</option>
        <option value="round10">無條件進位至 10 元整數</option>
        <option value="exact">保留小數點後 1 位</option>
      </select>

      <div style="font-size:13px;font-weight:600;margin-bottom:6px;">主揪收款備註 (銀行帳號 / LINE Pay / 街口)</div>
      <input type="text" id="input-payment-note" class="ios-input" placeholder="選填，會自動附在複製的分帳訊息底部">
    </div>
  </div>

  <!-- 分頁 4：每人應付結算 -->
  <div class="tab-pane" id="tab-result">
    <div class="section-header">
      <span>每人應付金額明細</span>
      <span id="result-diff-label"></span>
    </div>
    <div id="person-cards"></div>
    <button type="button" class="btn-primary" id="btn-copy-all" style="width:100%;background:var(--ios-green);margin-top:8px;">
      📋 一鍵複製完整 LINE 分帳明細
    </button>
    <button type="button" class="btn-primary" id="btn-share-sheet" style="width:100%;background:var(--ios-blue);margin-top:10px;">
      📤 呼叫 iPhone 系統分享選單
    </button>
  </div>

  <!-- 底部 iPhone Tab Bar -->
  <div class="tab-bar">
    <button type="button" class="tab-btn active" data-tab="tab-order">
      <span class="icon">🍽️</span><span>點餐輸入</span>
    </button>
    <button type="button" class="tab-btn" data-tab="tab-list">
      <span class="icon">🧾</span><span>餐點明細</span>
    </button>
    <button type="button" class="tab-btn" data-tab="tab-settings">
      <span class="icon">⚙️</span><span>費用設定</span>
    </button>
    <button type="button" class="tab-btn" data-tab="tab-result">
      <span class="icon">💰</span><span>分帳結算</span>
    </button>
  </div>

<script>
  const state = ${stateJson};
  let currentSplitMode = "individual";
  let selectedCustomMembers = new Set();

  function notifyScriptable(action, payload) {
    try {
      const encoded = encodeURIComponent(JSON.stringify(payload));
      const iframe = document.createElement("iframe");
      iframe.style.display = "none";
      iframe.src = "scriptable://" + action + "?data=" + encoded;
      document.body.appendChild(iframe);
      setTimeout(() => iframe.remove(), 300);
    } catch (e) {}
  }

  function showToast(msg) {
    const t = document.getElementById("toast");
    t.textContent = msg;
    t.classList.add("show");
    setTimeout(() => t.classList.remove("show"), 2200);
  }

  function syncAndRender() {
    notifyScriptable("save", state);
    render();
  }

  function calculateAll() {
    const memberCount = state.members.length;
    const stats = {};
    state.members.forEach(m => {
      stats[m] = { indItems: [], sharedItems: [], indTotal: 0, sharedTotal: 0, feeShare: 0, finalPay: 0, roundedPay: 0 };
    });

    let itemsTotal = 0;
    state.items.forEach(item => {
      const total = (Number(item.price) || 0) * (Number(item.qty) || 1);
      itemsTotal += total;
      if (item.mode === "individual" && stats[item.target]) {
        stats[item.target].indItems.push({ name: item.name, qty: item.qty, total });
        stats[item.target].indTotal += total;
      } else if (item.mode === "all" && memberCount > 0) {
        const per = total / memberCount;
        state.members.forEach(m => {
          stats[m].sharedItems.push({ name: item.name, count: memberCount, share: per });
          stats[m].sharedTotal += per;
        });
      } else if (item.mode === "custom" && Array.isArray(item.target)) {
        const valid = item.target.filter(m => state.members.includes(m));
        if (valid.length > 0) {
          const per = total / valid.length;
          valid.forEach(m => {
            stats[m].sharedItems.push({ name: item.name, count: valid.length, share: per });
            stats[m].sharedTotal += per;
          });
        }
      }
    });

    const serviceFee = (itemsTotal * (Number(state.serviceRate) || 0)) / 100;
    const netAdj = serviceFee + (Number(state.otherFee) || 0) - (Number(state.discount) || 0);
    const grandTotal = Math.max(0, itemsTotal + netAdj);

    let sumRounded = 0;
    state.members.forEach(m => {
      const sub = stats[m].indTotal + stats[m].sharedTotal;
      if (state.feeSplitRule === "proportional" && itemsTotal > 0) {
        stats[m].feeShare = netAdj * (sub / itemsTotal);
      } else {
        stats[m].feeShare = memberCount > 0 ? netAdj / memberCount : 0;
      }
      stats[m].finalPay = Math.max(0, sub + stats[m].feeShare);
      if (state.rounding === "round5") {
        stats[m].roundedPay = Math.ceil(stats[m].finalPay / 5) * 5;
      } else if (state.rounding === "round10") {
        stats[m].roundedPay = Math.ceil(stats[m].finalPay / 10) * 10;
      } else if (state.rounding === "exact") {
        stats[m].roundedPay = Math.round(stats[m].finalPay * 10) / 10;
      } else {
        stats[m].roundedPay = Math.round(stats[m].finalPay);
      }
      sumRounded += stats[m].roundedPay;
    });

    return { itemsTotal, serviceFee, grandTotal, sumRounded, stats };
  }

  function buildShareText(singleMember = null) {
    const res = calculateAll();
    const d = new Date();
    const dateStr = d.getFullYear() + "/" + (d.getMonth() + 1) + "/" + d.getDate();

    if (singleMember && res.stats[singleMember]) {
      const s = res.stats[singleMember];
      let txt = "🥢【聚餐分帳通知 - " + singleMember + "】\\n";
      txt += "📅 日期：" + dateStr + "\\n";
      txt += "💰 應付金額：NT$ " + s.roundedPay.toLocaleString() + "\\n";
      txt += "──────────────\\n";
      s.indItems.forEach(i => { txt += "• 個人：" + i.name + " $" + i.total + "\\n"; });
      s.sharedItems.forEach(i => { txt += "• 公攤：" + i.name + " (" + i.count + "人分) $" + Math.round(i.share) + "\\n"; });
      txt += "• 服務費與折扣攤提：$" + Math.round(s.feeShare) + "\\n";
      if (state.paymentNote) txt += "\\n📌 收款資訊：\\n" + state.paymentNote;
      return txt;
    }

    let txt = "🥢【聚餐分帳結算清單】\\n";
    txt += "📅 日期：" + dateStr + " (" + state.members.length + "人)\\n";
    txt += "🧾 帳單總計：NT$ " + Math.round(res.grandTotal).toLocaleString() + "\\n";
    txt += "──────────────\\n";
    state.members.forEach(m => {
      const s = res.stats[m];
      txt += "• " + m + "：NT$ " + s.roundedPay.toLocaleString() + " (個人$" + s.indTotal + " + 公攤$" + Math.round(s.sharedTotal) + " + 附加$" + Math.round(s.feeShare) + ")\\n";
    });
    if (state.paymentNote) txt += "\\n📌 收款資訊：\\n" + state.paymentNote;
    return txt;
  }

  function render() {
    const res = calculateAll();
    document.getElementById("top-grand-total").textContent = "NT$ " + Math.round(res.grandTotal).toLocaleString();
    document.getElementById("top-meta-members").textContent = "👥 " + state.members.length + " 人用餐";
    document.getElementById("top-meta-items").textContent = "🍽️ " + state.items.length + " 項餐點";
    document.getElementById("top-meta-fee").textContent = "🧾 服務費 " + state.serviceRate + "%";
    document.getElementById("member-count-label").textContent = state.members.length + " 人";

    // Render member chips
    const chipsEl = document.getElementById("member-chips");
    chipsEl.innerHTML = "";
    state.members.forEach(m => {
      const chip = document.createElement("div");
      chip.className = "member-chip";
      chip.innerHTML = '<span class="avatar">' + m.charAt(0) + '</span><span>' + m + '</span><button type="button" class="del" data-del="' + m + '">×</button>';
      chipsEl.appendChild(chip);
    });

    // Update select-member
    const sel = document.getElementById("select-member");
    const prevVal = sel.value;
    sel.innerHTML = "";
    state.members.forEach(m => {
      const opt = document.createElement("option");
      opt.value = m; opt.textContent = m;
      if (m === prevVal) opt.selected = true;
      sel.appendChild(opt);
    });

    // Update custom checks
    const checksEl = document.getElementById("custom-checks");
    checksEl.innerHTML = "";
    state.members.forEach(m => {
      const btn = document.createElement("button");
      btn.type = "button";
      btn.className = "check-pill" + (selectedCustomMembers.has(m) ? " selected" : "");
      btn.textContent = (selectedCustomMembers.has(m) ? "✓ " : "+ ") + m;
      btn.addEventListener("click", () => {
        if (selectedCustomMembers.has(m)) selectedCustomMembers.delete(m);
        else selectedCustomMembers.add(m);
        render();
      });
      checksEl.appendChild(btn);
    });

    // Render items list
    document.getElementById("list-subtotal").textContent = "小計 NT$ " + Math.round(res.itemsTotal).toLocaleString();
    const itemsEl = document.getElementById("items-container");
    itemsEl.innerHTML = "";
    if (state.items.length === 0) {
      itemsEl.innerHTML = '<div style="text-align:center;color:var(--ios-sub);padding:20px;font-size:13px;">尚無餐點品項</div>';
    } else {
      state.items.forEach(item => {
        const row = document.createElement("div");
        row.className = "item-row";
        const total = item.price * item.qty;
        let tag = "";
        if (item.mode === "individual") tag = "👤 個人：" + item.target;
        else if (item.mode === "all") tag = "👥 全員平分 (" + state.members.length + "人)";
        else tag = "🎯 分攤：" + (item.target || []).join("、");
        row.innerHTML = '<div><div class="item-title">' + item.name + (item.qty > 1 ? ' x' + item.qty : '') + '</div><div class="item-sub">' + tag + '</div></div><div class="item-right"><span class="item-price">NT$ ' + total.toLocaleString() + '</span><button type="button" class="btn-del-item" data-id="' + item.id + '">×</button></div>';
        itemsEl.appendChild(row);
      });
    }

    // Render person result cards
    const pCards = document.getElementById("person-cards");
    pCards.innerHTML = "";
    state.members.forEach(m => {
      const s = res.stats[m];
      const card = document.createElement("div");
      card.className = "person-card";
      const indHtml = s.indItems.length ? s.indItems.map(i => "• 個人：" + i.name + " ($" + i.total + ")").join("<br>") : "• 無個人單點";
      const shareHtml = s.sharedItems.length ? s.sharedItems.map(i => "• 公攤：" + i.name + " ($" + Math.round(i.share) + ")").join("<br>") : "• 無公攤品項";
      card.innerHTML = '<div class="person-top"><div class="person-name"><span style="width:26px;height:26px;border-radius:50%;background:var(--ios-orange);color:#fff;display:inline-flex;align-items:center;justify-content:center;font-size:13px;">' + m.charAt(0) + '</span>' + m + '</div><div class="person-amount">NT$ ' + s.roundedPay.toLocaleString() + '</div></div><div class="person-details">' + indHtml + '<br>' + shareHtml + '<br>• 服務費與折扣分攤：$' + Math.round(s.feeShare) + '</div><div class="person-actions"><button type="button" class="pill-btn pill-orange btn-copy-one" data-member="' + m + '">複製個人明細</button></div>';
      pCards.appendChild(card);
    });
  }

  // Event bindings
  document.getElementById("btn-add-member").addEventListener("click", () => {
    const inp = document.getElementById("member-input");
    const name = inp.value.trim();
    if (name && !state.members.includes(name)) {
      state.members.push(name);
      inp.value = "";
      syncAndRender();
      showToast("已新增：" + name);
    }
  });

  document.querySelectorAll("[data-preset]").forEach(btn => {
    btn.addEventListener("click", () => {
      const name = btn.dataset.preset;
      if (!state.members.includes(name)) {
        state.members.push(name);
        syncAndRender();
      }
    });
  });

  document.getElementById("member-chips").addEventListener("click", e => {
    const delBtn = e.target.closest("[data-del]");
    if (delBtn) {
      const name = delBtn.dataset.del;
      state.members = state.members.filter(m => m !== name);
      syncAndRender();
    }
  });

  document.querySelectorAll("[data-mode]").forEach(btn => {
    btn.addEventListener("click", () => {
      document.querySelectorAll("[data-mode]").forEach(b => b.classList.remove("active"));
      btn.classList.add("active");
      currentSplitMode = btn.dataset.mode;
      document.getElementById("box-individual").style.display = currentSplitMode === "individual" ? "block" : "none";
      document.getElementById("box-custom").style.display = currentSplitMode === "custom" ? "block" : "none";
    });
  });

  document.getElementById("btn-add-item").addEventListener("click", () => {
    const name = document.getElementById("item-name").value.trim();
    const price = parseFloat(document.getElementById("item-price").value) || 0;
    const qty = parseInt(document.getElementById("item-qty").value, 10) || 1;
    if (!name || price <= 0) { showToast("請輸入餐點名稱與金額"); return; }
    let target = "all";
    if (currentSplitMode === "individual") {
      target = document.getElementById("select-member").value;
    } else if (currentSplitMode === "custom") {
      target = Array.from(selectedCustomMembers);
      if (target.length === 0) { showToast("請點選分攤成員"); return; }
    }
    state.items.push({ id: "i_" + Date.now(), name, price, qty, mode: currentSplitMode, target });
    document.getElementById("item-name").value = "";
    document.getElementById("item-price").value = "";
    document.getElementById("item-qty").value = "1";
    syncAndRender();
    showToast("已加入：" + name);
  });

  document.getElementById("items-container").addEventListener("click", e => {
    const btn = e.target.closest("[data-id]");
    if (btn) {
      state.items = state.items.filter(i => i.id !== btn.dataset.id);
      syncAndRender();
    }
  });

  document.querySelectorAll(".fee-btn").forEach(btn => {
    btn.addEventListener("click", () => {
      document.querySelectorAll(".fee-btn").forEach(b => b.classList.remove("active"));
      btn.classList.add("active");
      if (btn.dataset.rate !== "custom") {
        state.serviceRate = parseFloat(btn.dataset.rate) || 0;
        document.getElementById("input-service-rate").value = state.serviceRate;
        syncAndRender();
      }
    });
  });

  document.getElementById("input-service-rate").addEventListener("input", e => {
    state.serviceRate = parseFloat(e.target.value) || 0;
    syncAndRender();
  });
  document.getElementById("input-discount").addEventListener("input", e => {
    state.discount = parseFloat(e.target.value) || 0;
    syncAndRender();
  });
  document.getElementById("input-other-fee").addEventListener("input", e => {
    state.otherFee = parseFloat(e.target.value) || 0;
    syncAndRender();
  });
  document.getElementById("select-fee-rule").addEventListener("change", e => {
    state.feeSplitRule = e.target.value;
    syncAndRender();
  });
  document.getElementById("select-rounding").addEventListener("change", e => {
    state.rounding = e.target.value;
    syncAndRender();
  });
  document.getElementById("input-payment-note").addEventListener("input", e => {
    state.paymentNote = e.target.value;
    syncAndRender();
  });

  // Tab navigation
  document.querySelectorAll(".tab-btn").forEach(btn => {
    btn.addEventListener("click", () => {
      document.querySelectorAll(".tab-btn").forEach(b => b.classList.remove("active"));
      document.querySelectorAll(".tab-pane").forEach(p => p.classList.remove("active"));
      btn.classList.add("active");
      document.getElementById(btn.dataset.tab).classList.add("active");
    });
  });

  // Copy & Share
  document.getElementById("btn-copy-all").addEventListener("click", () => {
    const text = buildShareText();
    notifyScriptable("copy", { text });
    showToast("已複製到 iPhone 剪貼簿！");
  });

  document.getElementById("btn-share-sheet").addEventListener("click", () => {
    const text = buildShareText();
    notifyScriptable("share", { text });
  });

  document.getElementById("btn-share-top").addEventListener("click", () => {
    const text = buildShareText();
    notifyScriptable("share", { text });
  });

  document.getElementById("person-cards").addEventListener("click", e => {
    const btn = e.target.closest(".btn-copy-one");
    if (btn) {
      const text = buildShareText(btn.dataset.member);
      notifyScriptable("copy", { text });
      showToast("已複製 " + btn.dataset.member + " 的分帳明細！");
    }
  });

  document.getElementById("btn-reset").addEventListener("click", () => {
    state.members = [];
    state.items = [];
    state.serviceRate = 0;
    state.discount = 0;
    state.otherFee = 0;
    syncAndRender();
    showToast("已清空資料");
  });

  document.getElementById("btn-sample").addEventListener("click", () => {
    Object.assign(state, ${ JSON.stringify(DEFAULT_STATE) });
    syncAndRender();
    showToast("已載入範例資料");
  });

  render();
<\/script>
</body>
</html>`;
}

// 主程式入口
async function main() {
  const currentState = loadStateFromDisk();

  // 若在 iOS 桌面 Widget 中執行
  if (config.runsInWidget) {
    const widget = buildWidget(currentState);
    Script.setWidget(widget);
    Script.complete();
    return;
  }

  // 在 Scriptable App 內或透過捷徑開啟互動式 iPhone 介面
  const wv = new WebView();
  await wv.loadHTML(buildIPhoneHTML(currentState));

  wv.shouldAllowRequest = (req) => {
    const url = req.url;
    if (!url.startsWith("scriptable://")) return true;
    try {
      const withoutScheme = url.replace("scriptable://", "");
      const parts = withoutScheme.split("?data=");
      const action = parts[0];
      const payload = parts[1] ? JSON.parse(decodeURIComponent(parts[1])) : {};

      if (action === "save") {
        saveStateToDisk(payload);
      } else if (action === "copy") {
        Pasteboard.copyString(payload.text || "");
      } else if (action === "share") {
        ShareSheet.present([payload.text || ""]);
      }
    } catch (e) {
      console.error("Bridge Error: " + e);
    }
    return false;
  };

  await wv.present(true);
  Script.complete();
}

await main();

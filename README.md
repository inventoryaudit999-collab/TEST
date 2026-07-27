# Trading Report Bakery v8 — GRP.68,78

แอปบันทึก Stock Count รายเดือน สำหรับแผนก Bakery GRP.68 & GRP.78  
CP Axtra (Makro) · Region 1 · **v8** (UOM + Variance Flags + Store Reference Band)

---

## 🔗 Firebase Project

| ค่า | รายละเอียด |
|-----|-----------|
| Project ID | `test-e907e` |
| Database URL | `https://test-e907e-default-rtdb.asia-southeast1.firebasedatabase.app` |
| Hosting URL | `https://test-e907e.web.app` |

---

## 🚀 Deploy ขึ้น Firebase Hosting

### ขั้นที่ 1 — ติดตั้ง Firebase CLI
```bash
npm install -g firebase-tools
```

### ขั้นที่ 2 — Login
```bash
firebase login
```

### ขั้นที่ 3 — Deploy (1 คำสั่ง)
```bash
firebase deploy --only hosting --project test-e907e
```

หลัง deploy สำเร็จ แอปพร้อมใช้ที่:
- **https://test-e907e.web.app**
- https://test-e907e.firebaseapp.com

---

## 📤 Push ขึ้น GitHub (optional)

```bash
git init
git add .
git commit -m "Trading Report Bakery v8 — test-e907e"
git branch -M main
git remote add origin https://github.com/<YOUR_USERNAME>/<REPO_NAME>.git
git push -u origin main
```

> GitHub Pages ไม่รองรับ Firebase Auth/Realtime DB โดยตรง  
> แนะนำ **Firebase Hosting** เป็นหลัก

---

## 🌱 Seed Demo Data

1. เปิดแอปใน browser
2. Login: `store001` / `welcome1` (Store 1 — ลาดพร้าว)
3. ไปที่ **Admin → Clear All**
4. กดปุ่ม **🌱 Seed ข้อมูลเดโม**

ข้อมูลจะถูกเขียนลง `demo/` namespace — ไม่กระทบ live data

---

## 📁 ไฟล์ในโปรเจกต์

| ไฟล์ | หน้าที่ |
|------|---------|
| `index.html` | UI หลัก |
| `firebase.js` | Firebase config (project: test-e907e) |
| `app.js` | Logic ทั้งหมด (v5 + UOM + Flags + Band) |
| `style.css` | Styles |
| `data.json` | Master items + stores (330 items, 185 stores) |
| `master_uom.json` | UOM reference (pack_size, packtype per item) |
| `master_cost.json` | Cost reference (display only) |
| `store_reference_band.json` | Cost band per store (avg/min/max จาก GL 6420006) |
| `seed.js` | Demo data seed script |

---

## 🗄️ Database Structure

ทุก read/write อยู่ใต้ `demo/` namespace

```
demo/
  entries/{storeNo}/{YYYY-MM}/{itemCode}
    qty:         12.5
    uom:         "ลัง"
    pack_size:   120
    counted_at:  1721740000000
  monthControl/{YYYY-MM}/active   true|false
  logs/{pushId}/...
  masterData/items
  masterData/stores
  presence/{storeNo}
```

---

## 🏷️ Demo Cases ที่ Seed ไว้ (Stores 1, เดือน 2026-06 & 07)

| # | Case | June | July | Flag |
|---|------|------|------|------|
| 1 | UOM change | ชิ้น qty 240 | ลัง qty 2 | ⚠️ หน่วยนับเปลี่ยน |
| 2 | Pack mismatch (item 859997) | ลัง / 6000 | ลัง / 16000 | ⚠️ ขนาดบรรจุไม่ตรง master |
| 3 | Large increase | ลัง qty 12 | ลัง qty 22 | 🔺 +83% |
| 4 | Large decrease | กล่อง qty 40 | กล่อง qty 15 | 🔻 −63% |
| 5 | Legacy bare number | 35 (number) | — | หน่วยไม่ระบุ |
| 6 | New item | — | ถุง qty 5 | 🆕 รายการใหม่ |
| 7 | Missing UOM | ถุง qty 8 | qty 9 / uom null | ⚠️ ยังไม่ระบุหน่วย (blocks save) |
| 8–20 | Stable (±5% drift) | same UOM | same UOM | ไม่มี flag |

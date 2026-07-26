# Trading Report Bakery v6 — GRP.68,78

แอปพลิเคชันบันทึก Stock Count รายเดือน สำหรับแผนก Bakery GRP.68 & GRP.78  
CP Axtra (Makro) · Region 1

## ✅ Firebase Project
- **Project:** `test-e907e`
- **Database:** `https://test-e907e-default-rtdb.asia-southeast1.firebasedatabase.app`
- **Hosting:** Deploy via Firebase CLI

---

## 🚀 วิธี Deploy ขึ้น Firebase Hosting

### ขั้นที่ 1 — ติดตั้ง Firebase CLI
```bash
npm install -g firebase-tools
```

### ขั้นที่ 2 — Login Firebase
```bash
firebase login
```

### ขั้นที่ 3 — Deploy
```bash
firebase deploy --only hosting --project test-e907e
```

URL จะแสดงหลัง deploy เสร็จ เช่น:  
`https://test-e907e.web.app`

---

## 🌱 Seed ข้อมูลเดโม

หลัง login เข้าแอป ให้ไปที่ **Admin → Clear All** แล้วกด:  
**🌱 Seed ข้อมูลเดโม**

จะเขียน demo data ลงใน path `demo/` ใน Firebase Realtime Database  
(ไม่กระทบ live data เพราะแยก namespace)

บัญชีเดโม: `store001` / `welcome1` (Store 1 — ลาดพร้าว)

---

## 📁 โครงสร้างไฟล์

```
index.html        — UI หลัก
firebase.js       — Firebase config (project: test-e907e)
app.js            — Logic ทั้งหมด (v5 + UOM features)
style.css         — Styles
data.json         — Master items + stores (330 items, 185 stores)
master_uom.json   — UOM reference (pack_size, packtype per item)
master_cost.json  — Cost reference (display only, never written)
seed.js           — Demo data seed script
firebase.json     — Firebase Hosting config
.firebaserc       — Firebase project alias
```

---

## ⚙️ Database Namespace

ทุก read/write อยู่ใต้ `demo/` เท่านั้น — ป้องกัน live data

```
demo/entries/{storeNo}/{YYYY-MM}/{itemCode}
demo/monthControl/{YYYY-MM}/active
demo/logs/
```

---

## 📋 Demo Cases (Seed ไว้แล้ว)

| # | Case | Flag ที่คาดหวัง |
|---|------|----------------|
| 1 | UOM change (ชิ้น→ลัง) | ⚠️ หน่วยนับเปลี่ยน |
| 2 | Pack size mismatch (item 859997) | ⚠️ ขนาดบรรจุไม่ตรงกับ master |
| 3 | Large increase +83% | 🔺 +83% |
| 4 | Large decrease −63% | 🔻 −63% |
| 5 | Legacy bare number | หน่วยไม่ระบุ |
| 6 | New item (no prior month) | 🆕 รายการใหม่ |
| 7 | Missing UOM (blocks save) | ⚠️ ยังไม่ระบุหน่วยนับ |
| 8–20 | Stable items (±5% drift) | ไม่มี flag |

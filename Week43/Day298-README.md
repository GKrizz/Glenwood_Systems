
# Fix for `userId` Conversion Error (MIPS Flowsheet / Auth API)

---

# 🎯 Objective

Resolve backend error:

> ❌ API fails when `userId=""`
> ✅ Ensure MIPS Flowsheet loads correctly

---

# 🚨 Issue Summary

### 🔗 API

```http id="4k3z0u"
GET /glaceemr_backend_beta_v2/calvary/api/desktop/auth/get/username?userId=
```

### ❌ Response

```json id="m9jv1a"
{
  "success": false,
  "errorMessage": "Failed to convert value of type 'java.lang.String' to required type 'int'; For input string: \"\""
}
```

---

# 🧠 Root Cause

| Problem                     | Explanation                |
| --------------------------- | -------------------------- |
| `userId` sent as `""`       | Empty string from frontend |
| Backend expects `int`       | Spring tries to parse      |
| `"" → int` conversion fails | Throws exception           |

👉 Spring Boot behavior:

```java id="3o1i0x"
@RequestParam int userId
```

➡️ Automatically tries:

```text id="k9s6c2"
Integer.parseInt("")
```

➡️ ❌ Exception

---

# 🔥 Impact

* Authentication API fails
* MIPS Flowsheet fails to load
* UI may silently break

---

# 🛠️ Fix Strategy

---

# ✅ 1️⃣ FRONTEND FIX (PRIMARY FIX)

## ❌ Current

```javascript id="z0p8wr"
userId: ""
```

---

## ✅ Fix

```javascript id="h1kz3v"
if (userId) {
   params.userId = userId;
}
```

OR

```javascript id="r3y6nc"
params.userId = userId || null;
```

OR (best)

```javascript id="t8x5qp"
if (userId !== undefined && userId !== null && userId !== '') {
    params.userId = userId;
}
```

---

# ✅ 2️⃣ BACKEND FIX (DEFENSIVE – MUST DO)

## ❌ Current Controller

```java id="k2a9df"
@RequestParam int userId
```

---

## ✅ Fix Option A (Recommended)

```java id="n7v3wx"
@RequestParam(required = false) Integer userId
```

---

## ✅ Fix Option B (Safe Parsing)

```java id="q6c2mv"
@RequestParam(required = false) String userId
```

Then:

```java id="p9s8ae"
Integer parsedUserId = null;

if (userId != null && !userId.trim().isEmpty()) {
    parsedUserId = Integer.parseInt(userId);
}
```

---

## ✅ Fix Option C (Default Value)

```java id="y2x1ld"
@RequestParam(defaultValue = "-1") int userId
```

---

# ✅ 3️⃣ GLOBAL EXCEPTION HANDLING (BEST PRACTICE)

```java id="d4m7zk"
@ExceptionHandler(MethodArgumentTypeMismatchException.class)
public ResponseEntity<?> handleTypeMismatch(Exception ex) {
    return ResponseEntity.badRequest().body("Invalid userId");
}
```

---

# 🧪 Validation

---

## ✅ Case 1: Valid userId

```http id="5u0bqj"
?userId=123
```

✔ Works

---

## ✅ Case 2: Empty userId

```http id="2n4f7k"
?userId=
```

✔ Should NOT crash
✔ Should return controlled response

---

## ✅ Case 3: Missing param

```http id="m3h8va"
(no userId)
```

✔ Should work

---

# 🔒 Recommended Final Design

| Layer      | Rule                    |
| ---------- | ----------------------- |
| Frontend   | Never send empty string |
| Backend    | Never trust input       |
| Controller | Use Integer not int     |
| Validation | Handle null safely      |

---

# 📌 Final Fix Summary

```text id="z1h7xn"
Frontend: remove empty userId
Backend: change int → Integer
Validation: safe parsing
```

---

# 🏁 Final Outcome

| Before                 | After             |
| ---------------------- | ----------------- |
| API crash ❌            | Safe handling ✅   |
| success:false ❌        | success:true ✅    |
| MIPS Flowsheet fails ❌ | Loads correctly ✅ |

---

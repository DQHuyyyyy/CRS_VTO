# 🎯 Đề Xuất: Xử Lý Negative Constraints & Natural Language Understanding

## 📋 Vấn Đề Hiện Tại

**Scenario:**
```
User Query 1: "black hats"
→ System: Trả về sản phẩm mũ màu đen ✅

User Query 2: "i dislike this color"
→ System: Không hiểu "this color" = màu đen ❌
→ System: Không loại bỏ màu đen khỏi kết quả ❌
```

**Nguyên nhân:**
1. ❌ Không có **negative constraint parsing** (dislike, not, exclude)
2. ❌ Không có **context reference resolution** ("this color" = màu gì?)
3. ❌ Không có **exclusion mechanism** trong filtering logic
4. ❌ Chỉ parse **positive constraints** (color, material, theme)

---

## 🎯 Yêu Cầu

- ✅ Xử lý negative constraints cho **Color, Material, Theme**
- ✅ Hiểu ngôn ngữ tự nhiên: "dislike", "not", "exclude", "remove"
- ✅ Resolve context references: "this color", "that material", "these themes"
- ✅ Tích hợp vào pipeline hiện tại
- ✅ Có thể mở rộng cho các attributes khác

---

## 💡 Phương Pháp Đề Xuất: Hybrid Approach

### **Tổng Quan: 3-Layer Architecture**

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: Context Reference Resolution                   │
│  - Resolve "this/that/these" → actual values             │
│  - Track conversation history                            │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 2: Constraint Extraction (LLM + Rule-based)       │
│  - Extract positive constraints                         │
│  - Extract negative constraints (NEW)                    │
│  - Extract attribute types (color/material/theme)       │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 3: Filter Application                            │
│  - Apply positive filters                               │
│  - Apply negative filters (exclusion) (NEW)             │
└─────────────────────────────────────────────────────────┘
```

---

## 🔍 Layer 1: Context Reference Resolution

### **1.1. Conversation History Tracking**

**Mục đích:** Lưu trữ context từ các query trước để resolve references.

**Data Structure:**
```typescript
interface ConversationContext {
  // Current query
  currentQuery: string;
  
  // Previous queries (last N queries)
  history: Array<{
    query: string;
    timestamp: Date;
    extractedConstraints: {
      positive: {
        color?: string[];
        material?: string[];
        theme?: string[];
        category?: string;
        sex?: string;
        price?: { min?: number; max?: number };
      };
      negative: {
        color?: string[];
        material?: string[];
        theme?: string[];
      };
    };
    results?: {
      products: Product[];
      displayedAttributes: {
        colors: string[];
        materials: string[];
        themes: string[];
      };
    };
  }>;
  
  // Current active filters
  activeFilters: {
    positive: {...};
    negative: {...};
  };
}
```

**Ví dụ:**
```typescript
// Query 1: "black hats"
history[0] = {
  query: "black hats",
  extractedConstraints: {
    positive: { color: ["black"], category: "hats" },
    negative: {}
  },
  results: {
    displayedAttributes: {
      colors: ["black"],
      materials: ["cotton", "wool"],
      themes: ["casual"]
    }
  }
}

// Query 2: "i dislike this color"
// "this color" → resolve từ history[0].displayedAttributes.colors[0] = "black"
```

---

### **1.2. Reference Resolution Patterns**

**Patterns cần nhận diện:**

| Pattern | Example | Resolution |
|---------|---------|------------|
| **"this [attribute]"** | "this color", "this material" | Lấy từ query gần nhất |
| **"that [attribute]"** | "that color", "that material" | Lấy từ query trước đó |
| **"these [attributes]"** | "these colors", "these themes" | Lấy tất cả từ query gần nhất |
| **"the [attribute]"** | "the color", "the material" | Lấy từ query gần nhất |
| **"[attribute] from before"** | "color from before" | Lấy từ query trước đó |
| **"[attribute] in results"** | "color in results" | Lấy từ kết quả hiện tại |

**Algorithm:**
```
1. Detect reference pattern trong query
2. Identify attribute type (color/material/theme)
3. Look up trong conversation history:
   - "this/these/the" → history[-1] (most recent)
   - "that/those" → history[-2] (previous)
   - "from before" → history[-2] or earlier
4. Extract values từ:
   - extractedConstraints.positive[attribute]
   - results.displayedAttributes[attribute]
5. Return resolved values
```

**Ví dụ Implementation Logic:**
```typescript
function resolveReference(
  query: string,
  attribute: "color" | "material" | "theme",
  history: ConversationContext["history"]
): string[] {
  const lowerQuery = query.toLowerCase();
  
  // Pattern: "this [attribute]"
  if (lowerQuery.match(new RegExp(`this ${attribute}`, "i"))) {
    const lastQuery = history[history.length - 1];
    return lastQuery?.extractedConstraints.positive[attribute] || 
           lastQuery?.results?.displayedAttributes[attribute] || [];
  }
  
  // Pattern: "that [attribute]"
  if (lowerQuery.match(new RegExp(`that ${attribute}`, "i"))) {
    const prevQuery = history[history.length - 2];
    return prevQuery?.extractedConstraints.positive[attribute] || 
           prevQuery?.results?.displayedAttributes[attribute] || [];
  }
  
  // Pattern: "these [attributes]" (plural)
  if (lowerQuery.match(new RegExp(`these ${attribute}s?`, "i"))) {
    const lastQuery = history[history.length - 1];
    return lastQuery?.results?.displayedAttributes[attribute] || [];
  }
  
  // ... more patterns
  
  return [];
}
```

---

## 🧠 Layer 2: Constraint Extraction (LLM + Rule-based)

### **2.1. Hybrid Approach: LLM Primary + Rule-based Fallback**

**Tại sao Hybrid?**
- ✅ **LLM**: Hiểu ngữ cảnh phức tạp, linh hoạt
- ✅ **Rule-based**: Nhanh, reliable, không tốn phí
- ✅ **Fallback**: Rule-based khi LLM chậm/lỗi

---

### **2.2. LLM-based Constraint Extraction**

**Mục đích:** Dùng Qwen/GPT để extract constraints từ natural language.

**Prompt Template:**
```
System: Bạn là một hệ thống trích xuất constraints từ câu người dùng cho hệ thống tư vấn thời trang.

Nhiệm vụ:
1. Trích xuất POSITIVE constraints (người dùng MUỐN):
   - color: ["black", "white", ...]
   - material: ["cotton", "wool", ...]
   - theme: ["casual", "formal", ...]
   - category: "hats", "shirts", ...
   - sex: "men", "women", ...
   - price: {min: 10, max: 50}

2. Trích xuất NEGATIVE constraints (người dùng KHÔNG MUỐN):
   - exclude_color: ["black", "red", ...]
   - exclude_material: ["synthetic", ...]
   - exclude_theme: ["formal", ...]

3. Xác định intent:
   - "add_filter": Thêm filter mới
   - "remove_filter": Loại bỏ filter
   - "refine_search": Tinh chỉnh tìm kiếm

Input: "{user_query}"
Context: {conversation_context}

Output JSON format:
{
  "intent": "add_filter" | "remove_filter" | "refine_search",
  "positive": {
    "color": ["black"],
    "material": [],
    "theme": [],
    "category": "hats",
    "sex": null,
    "price": null
  },
  "negative": {
    "exclude_color": [],
    "exclude_material": [],
    "exclude_theme": []
  }
}
```

**Implementation:**
```typescript
async function extractConstraintsLLM(
  query: string,
  context: ConversationContext
): Promise<ExtractedConstraints> {
  // Try Qwen first
  try {
    const baseUrl = process.env.RETRIEVAL_API_URL ?? "http://127.0.0.1:8000";
    const url = `${baseUrl}/qwen/extract-constraints`;
    
    const res = await fetch(url, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({
        query,
        context: {
          lastQuery: context.history[context.history.length - 1],
          activeFilters: context.activeFilters
        }
      }),
      signal: AbortSignal.timeout(5000) // 5s timeout
    });
    
    if (res.ok) {
      const data = await res.json();
      return data.constraints;
    }
  } catch (error) {
    console.warn("[CONSTRAINTS] LLM extraction failed, using rule-based");
  }
  
  // Fallback to rule-based
  return extractConstraintsRuleBased(query, context);
}
```

---

### **2.3. Rule-based Constraint Extraction (Fallback)**

**Mục đích:** Parse constraints bằng regex/patterns khi LLM không available.

**Negative Constraint Patterns:**

| Pattern | Example | Extracted |
|---------|---------|-----------|
| **"dislike [value]"** | "dislike black" | exclude_color: ["black"] |
| **"not [value]"** | "not cotton" | exclude_material: ["cotton"] |
| **"exclude [value]"** | "exclude formal" | exclude_theme: ["formal"] |
| **"remove [value]"** | "remove red" | exclude_color: ["red"] |
| **"without [value]"** | "without synthetic" | exclude_material: ["synthetic"] |
| **"no [value]"** | "no black" | exclude_color: ["black"] |
| **"avoid [value]"** | "avoid cotton" | exclude_material: ["cotton"] |
| **"hate [value]"** | "hate formal" | exclude_theme: ["formal"] |
| **"don't want [value]"** | "don't want red" | exclude_color: ["red"] |
| **"don't like [value]"** | "don't like cotton" | exclude_material: ["cotton"] |

**Implementation:**
```typescript
function extractConstraintsRuleBased(
  query: string,
  context: ConversationContext
): ExtractedConstraints {
  const lowerQuery = query.toLowerCase();
  const constraints: ExtractedConstraints = {
    intent: "add_filter",
    positive: {},
    negative: {}
  };
  
  // Resolve references first
  const resolvedColor = resolveReference(query, "color", context.history);
  const resolvedMaterial = resolveReference(query, "material", context.history);
  const resolvedTheme = resolveReference(query, "theme", context.history);
  
  // Negative constraints patterns
  const negativePatterns = [
    { pattern: /dislike\s+(this|that|the|these|those)?\s*(color|material|theme)?\s*:?\s*(\w+)/gi, type: "general" },
    { pattern: /(don'?t|do not)\s+(want|like|need)\s+(\w+)/gi, type: "general" },
    { pattern: /exclude\s+(\w+)/gi, type: "general" },
    { pattern: /remove\s+(\w+)/gi, type: "general" },
    { pattern: /without\s+(\w+)/gi, type: "general" },
    { pattern: /no\s+(\w+)/gi, type: "general" },
    { pattern: /avoid\s+(\w+)/gi, type: "general" },
    { pattern: /hate\s+(\w+)/gi, type: "general" },
  ];
  
  // Color-specific negative patterns
  const colorNegativePatterns = [
    /(dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(this|that|the)?\s*color/gi,
    /(dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(\w+)\s+color/gi,
  ];
  
  // Material-specific negative patterns
  const materialNegativePatterns = [
    /(dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(this|that|the)?\s*material/gi,
    /(dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(\w+)\s+material/gi,
  ];
  
  // Theme-specific negative patterns
  const themeNegativePatterns = [
    /(dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(this|that|the)?\s*theme/gi,
    /(dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(\w+)\s+theme/gi,
  ];
  
  // Extract negative color constraints
  if (colorNegativePatterns.some(p => p.test(lowerQuery))) {
    if (resolvedColor.length > 0) {
      constraints.negative.exclude_color = resolvedColor;
    } else {
      // Try to extract color value directly
      const colorMatch = lowerQuery.match(/(?:dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(\w+)\s+color/);
      if (colorMatch) {
        constraints.negative.exclude_color = [colorMatch[1]];
      }
    }
  }
  
  // Extract negative material constraints
  if (materialNegativePatterns.some(p => p.test(lowerQuery))) {
    if (resolvedMaterial.length > 0) {
      constraints.negative.exclude_material = resolvedMaterial;
    } else {
      const materialMatch = lowerQuery.match(/(?:dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(\w+)\s+material/);
      if (materialMatch) {
        constraints.negative.exclude_material = [materialMatch[1]];
      }
    }
  }
  
  // Extract negative theme constraints
  if (themeNegativePatterns.some(p => p.test(lowerQuery))) {
    if (resolvedTheme.length > 0) {
      constraints.negative.exclude_theme = resolvedTheme;
    } else {
      const themeMatch = lowerQuery.match(/(?:dislike|don'?t like|hate|avoid|exclude|remove|no|without)\s+(\w+)\s+theme/);
      if (themeMatch) {
        constraints.negative.exclude_theme = [themeMatch[1]];
      }
    }
  }
  
  // Extract positive constraints (existing logic)
  // ... (keep existing parseConstraints logic)
  
  return constraints;
}
```

---

## 🔧 Layer 3: Filter Application

### **3.1. Extended Filter Structure**

**Current Structure:**
```typescript
type Filters = {
  color?: string;
  material?: string;
  theme?: string;
  category?: string;
  sex?: string;
  priceMin?: number;
  priceMax?: number;
  ratingMin?: number;
  ratingMax?: number;
}
```

**New Structure:**
```typescript
type Filters = {
  // Positive constraints (include)
  color?: string | string[];
  material?: string | string[];
  theme?: string | string[];
  category?: string;
  sex?: string;
  priceMin?: number;
  priceMax?: number;
  ratingMin?: number;
  ratingMax?: number;
  
  // Negative constraints (exclude) - NEW
  excludeColor?: string | string[];
  excludeMaterial?: string | string[];
  excludeTheme?: string | string[];
}
```

---

### **3.2. Filter Application Logic**

**Backend (Python - retrieval_service/main.py):**

```python
def pass_filters(idx: int, slots: Slots) -> bool:
    """Check if product passes all filters (positive + negative)."""
    if _meta is None or idx < 0 or idx >= len(_meta):
        return False
    
    m = _meta[idx]
    
    # === POSITIVE FILTERS (existing logic) ===
    # Category, Sex, Color, Material, Theme, Price, Rating
    # ... (keep existing logic)
    
    # === NEGATIVE FILTERS (NEW) ===
    
    # Exclude Color
    if slots.exclude_color:
        exclude_colors = slots.exclude_color if isinstance(slots.exclude_color, list) else [slots.exclude_color]
        product_color = _normalize_text(m.get("Color", ""))
        for exclude_color in exclude_colors:
            exclude_color_norm = _normalize_text(exclude_color)
            if exclude_color_norm in product_color or product_color in exclude_color_norm:
                return False  # Excluded!
    
    # Exclude Material
    if slots.exclude_material:
        exclude_materials = slots.exclude_material if isinstance(slots.exclude_material, list) else [slots.exclude_material]
        product_material = _normalize_text(m.get("Material", ""))
        for exclude_material in exclude_materials:
            exclude_material_norm = _normalize_text(exclude_material)
            if exclude_material_norm in product_material or product_material in exclude_material_norm:
                return False  # Excluded!
    
    # Exclude Theme
    if slots.exclude_theme:
        exclude_themes = slots.exclude_theme if isinstance(slots.exclude_theme, list) else [slots.exclude_theme]
        product_theme = _normalize_text(m.get("Theme", ""))
        for exclude_theme in exclude_themes:
            exclude_theme_norm = _normalize_text(exclude_theme)
            if exclude_theme_norm in product_theme or product_theme in exclude_theme_norm:
                return False  # Excluded!
    
    return True  # Passed all filters
```

**Frontend (TypeScript - components/chat-interface.tsx):**

```typescript
function applyClientFilters(
  products: Product[],
  filters: Filters
): Product[] {
  return products.filter((product) => {
    // === POSITIVE FILTERS (existing logic) ===
    // ... (keep existing logic)
    
    // === NEGATIVE FILTERS (NEW) ===
    
    // Exclude Color
    if (filters.excludeColor) {
      const excludeColors = Array.isArray(filters.excludeColor) 
        ? filters.excludeColor 
        : [filters.excludeColor];
      const productColor = product.color?.toLowerCase() || "";
      if (excludeColors.some(exclude => 
        productColor.includes(exclude.toLowerCase()) || 
        exclude.toLowerCase().includes(productColor)
      )) {
        return false; // Excluded!
      }
    }
    
    // Exclude Material
    if (filters.excludeMaterial) {
      const excludeMaterials = Array.isArray(filters.excludeMaterial) 
        ? filters.excludeMaterial 
        : [filters.excludeMaterial];
      const productMaterial = product.material?.toLowerCase() || "";
      if (excludeMaterials.some(exclude => 
        productMaterial.includes(exclude.toLowerCase()) || 
        exclude.toLowerCase().includes(productMaterial)
      )) {
        return false; // Excluded!
      }
    }
    
    // Exclude Theme
    if (filters.excludeTheme) {
      const excludeThemes = Array.isArray(filters.excludeTheme) 
        ? filters.excludeTheme 
        : [filters.excludeTheme];
      const productTheme = product.theme?.toLowerCase() || "";
      if (excludeThemes.some(exclude => 
        productTheme.includes(exclude.toLowerCase()) || 
        exclude.toLowerCase().includes(productTheme)
      )) {
        return false; // Excluded!
      }
    }
    
    return true; // Passed all filters
  });
}
```

---

## 🔄 Integration vào Pipeline Hiện Tại

### **Step 1: Update Frontend Context Management**

```typescript
// components/chat-interface.tsx

const [context, setContext] = useState<{
  baseQuery: string | null;
  filters: Filters; // Extended với excludeColor, excludeMaterial, excludeTheme
  history: ConversationContext["history"]; // NEW
}>({
  baseQuery: null,
  filters: {},
  history: []
});
```

### **Step 2: Update Constraint Parsing**

```typescript
// Replace parseConstraints() với extractConstraints()
async function extractConstraints(
  query: string,
  context: ConversationContext
): Promise<ExtractedConstraints> {
  // Try LLM first
  try {
    return await extractConstraintsLLM(query, context);
  } catch {
    // Fallback to rule-based
    return extractConstraintsRuleBased(query, context);
  }
}
```

### **Step 3: Update handleSendMessage()**

```typescript
const handleSendMessage = async () => {
  // ... existing code ...
  
  // Extract constraints (positive + negative)
  const extracted = await extractConstraints(input, {
    currentQuery: input,
    history: context.history,
    activeFilters: context.filters
  });
  
  // Resolve references
  const resolvedConstraints = resolveReferences(extracted, context.history);
  
  // Update context
  const nextContext = {
    baseQuery: context.baseQuery ?? input,
    filters: {
      ...context.filters,
      // Merge positive constraints
      ...resolvedConstraints.positive,
      // Merge negative constraints
      excludeColor: [
        ...(context.filters.excludeColor || []),
        ...(resolvedConstraints.negative.exclude_color || [])
      ],
      excludeMaterial: [
        ...(context.filters.excludeMaterial || []),
        ...(resolvedConstraints.negative.exclude_material || [])
      ],
      excludeTheme: [
        ...(context.filters.excludeTheme || []),
        ...(resolvedConstraints.negative.exclude_theme || [])
      ]
    },
    history: [
      ...context.history,
      {
        query: input,
        timestamp: new Date(),
        extractedConstraints: resolvedConstraints,
        results: { /* will be filled after API call */ }
      }
    ]
  };
  
  setContext(nextContext);
  
  // Build slots for API
  const slots = {
    category: nextContext.filters.category,
    sex: nextContext.filters.sex,
    color: nextContext.filters.color,
    material: nextContext.filters.material,
    theme: nextContext.filters.theme,
    price: nextContext.filters.priceMin || nextContext.filters.priceMax 
      ? { min: nextContext.filters.priceMin, max: nextContext.filters.priceMax }
      : undefined,
    rating: nextContext.filters.ratingMin || nextContext.filters.ratingMax
      ? { min: nextContext.filters.ratingMin, max: nextContext.filters.ratingMax }
      : undefined,
    // NEW: Negative constraints
    exclude_color: nextContext.filters.excludeColor,
    exclude_material: nextContext.filters.excludeMaterial,
    exclude_theme: nextContext.filters.excludeTheme
  };
  
  // ... rest of existing code ...
};
```

### **Step 4: Update Backend Slots Model**

```python
# retrieval_service/main.py

class Slots(BaseModel):
    category: Optional[str] = None
    sex: Optional[str] = None
    color: Optional[List[str]] = None
    material: Optional[List[str]] = None
    theme: Optional[List[str]] = None
    price: Optional[PriceRange] = None
    rating: Optional[RatingRange] = None
    # NEW: Negative constraints
    exclude_color: Optional[List[str]] = None
    exclude_material: Optional[List[str]] = None
    exclude_theme: Optional[List[str]] = None
```

---

## 📊 Example Flow

### **Scenario: "black hats" → "i dislike this color"**

```
┌─────────────────────────────────────────────────────────┐
│ Query 1: "black hats"                                   │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Extract Constraints                            │
│ - positive: { color: ["black"], category: "hats" }      │
│ - negative: {}                                          │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Apply Filters                                  │
│ - Include: color="black", category="hats"               │
│ - Exclude: none                                         │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Results: Black hats                                     │
│ - Displayed attributes: { colors: ["black"], ... }      │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Update Context History                                  │
│ history[0] = {                                           │
│   query: "black hats",                                   │
│   extractedConstraints: { ... },                         │
│   results: { displayedAttributes: { colors: ["black"] } }│
│ }                                                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Query 2: "i dislike this color"                         │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Resolve Reference                              │
│ - "this color" → history[0].displayedAttributes.colors  │
│ - Resolved: ["black"]                                    │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Extract Constraints                            │
│ - positive: {}                                           │
│ - negative: { exclude_color: ["black"] }                 │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Apply Filters                                  │
│ - Include: category="hats" (from previous)             │
│ - Exclude: color="black"                                 │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Results: Hats (NOT black)                               │
│ - All colors except black                                │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 Tóm Tắt

### **Các Component Cần Thêm:**

1. **ConversationContext** - Track history và context
2. **Reference Resolution** - Resolve "this/that/these"
3. **LLM Constraint Extraction** - Extract positive + negative constraints
4. **Rule-based Fallback** - Parse constraints bằng patterns
5. **Extended Filters** - Support excludeColor, excludeMaterial, excludeTheme
6. **Backend Filter Logic** - Apply negative filters trong pass_filters()

### **Lợi Ích:**

- ✅ Hiểu ngôn ngữ tự nhiên: "dislike", "not", "exclude"
- ✅ Resolve context references: "this color", "that material"
- ✅ Support negative constraints cho Color, Material, Theme
- ✅ Hybrid approach: LLM + Rule-based (reliable + fast)
- ✅ Backward compatible: Không ảnh hưởng logic hiện tại

### **Next Steps (Khi implement):**

1. Implement Layer 1: Reference Resolution
2. Implement Layer 2: Constraint Extraction (LLM + Rule-based)
3. Update Layer 3: Filter Application (Backend + Frontend)
4. Add new endpoint: `/qwen/extract-constraints` (optional)
5. Test với các scenarios khác nhau

---

**Tài liệu này mô tả đầy đủ phương pháp xử lý negative constraints và natural language understanding.**

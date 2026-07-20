# Component Registry trong Runtime UI: Thiết kế một contract ổn định giữa JSON và source code

## Bối cảnh

Trong kiến trúc Runtime UI, cấu trúc giao diện thường được mô tả bằng configuration:

```json
{
  "sections": [
    {
      "id": "main-hero",
      "type": "hero",
      "props": {
        "title": "Summer Campaign"
      }
    }
  ]
}
```

Frontend nhận `type`, tìm component tương ứng và render.

Một implementation cơ bản có thể là:

```ts
const registry = {
  hero: HeroSection,
  campaign: CampaignSection,
  productGrid: ProductGridSection,
};
```

```tsx
const Component = registry[section.type];

return <Component {...section.props} />;
```

Cách tiếp cận này đủ cho prototype, nhưng chưa mô tả đầy đủ vai trò của Component Registry trong một runtime production-ready.

Registry không chỉ là lookup table.

Nó là contract giữa:

```text
CMS / Backend Configuration
            ↓
     Runtime Renderer
            ↓
   Frontend Implementation
```

---

## Vấn đề khi configuration biết quá nhiều

Một thiết kế không nên sử dụng internal component name hoặc source path trong configuration:

```json
{
  "component": "src/features/home/HeroSectionV4"
}
```

Cách này tạo ra nhiều vấn đề:

* Configuration phụ thuộc trực tiếp vào cấu trúc source code.
* Refactor frontend có thể phá các config đã publish.
* Backend biết implementation detail không thuộc trách nhiệm của nó.
* Khó giới hạn component nào được runtime phép sử dụng.
* Khó kiểm soát props và schema của từng component.
* Có nguy cơ mở rộng runtime thành cơ chế gọi arbitrary implementation.

Configuration nên chỉ biết một identifier ổn định:

```json
{
  "type": "hero"
}
```

Identifier này thuộc public contract của runtime, không phải internal implementation detail.

---

## Registry như một public API

Một component registry có thể được xem như API do frontend cung cấp:

```ts
const sectionRegistry = {
  hero: HeroSection,
  campaign: CampaignSection,
  productGrid: ProductGridSection,
};
```

Public identifiers:

```text
hero
campaign
productGrid
```

Internal implementations:

```text
HeroSection
CampaignSection
ProductGridSection
```

Frontend có thể thay đổi implementation:

```ts
const sectionRegistry = {
  hero: HeroSectionV2,
};
```

Trong khi configuration vẫn giữ nguyên:

```json
{
  "type": "hero"
}
```

Đây là lợi ích cốt lõi của abstraction boundary.

---

## Registry như một whitelist

Không phải component nào trong frontend cũng nên được runtime sử dụng.

Ví dụ, runtime không nên tự do truy cập:

* Internal form controls.
* Authentication components.
* Payment implementation.
* Debugging tools.
* Components chưa ổn định.
* Components phụ thuộc sâu vào local application state.

Registry chỉ công khai những component đã được thiết kế để hoạt động trong runtime.

```ts
const sectionRegistry = {
  hero: HeroSection,
  campaign: CampaignSection,
  productGrid: ProductGridSection,
} as const;
```

Mọi `type` không có trong registry phải được từ chối hoặc xử lý bằng fallback.

```tsx
const entry = sectionRegistry[section.type];

if (!entry) {
  return <UnsupportedSection sectionType={section.type} />;
}
```

---

## Thiết kế type-safe registry

Một registry đơn giản thường làm mất type safety khi mỗi component nhận props khác nhau.

Có thể định nghĩa một map giữa section type và props:

```ts
type SectionPropsMap = {
  hero: {
    title: string;
    description?: string;
    variant: "default" | "campaign";
  };

  campaign: {
    campaignId: string;
  };

  productGrid: {
    categoryId: string;
    limit?: number;
  };
};
```

Sau đó tạo registry type:

```ts
import type { ComponentType } from "react";

type SectionRegistry = {
  [Type in keyof SectionPropsMap]: ComponentType<SectionPropsMap[Type]>;
};
```

Implementation:

```ts
const sectionRegistry: SectionRegistry = {
  hero: HeroSection,
  campaign: CampaignSection,
  productGrid: ProductGridSection,
};
```

Runtime section có thể được sinh từ cùng một map:

```ts
type RuntimeSection = {
  [Type in keyof SectionPropsMap]: {
    id: string;
    type: Type;
    props: SectionPropsMap[Type];
  };
}[keyof SectionPropsMap];
```

Cách này giúp giảm khả năng registry, component props và runtime config contract phát triển độc lập rồi mất đồng bộ.

Tuy nhiên, TypeScript chỉ bảo vệ compile-time.

Configuration từ network vẫn cần runtime validation.

---

## Registry kết hợp với schema validation

Một registry production-ready có thể lưu cả component và schema:

```ts
const sectionRegistry = {
  hero: {
    component: HeroSection,
    schema: HeroSectionSchema,
  },

  campaign: {
    component: CampaignSection,
    schema: CampaignSectionSchema,
  },

  productGrid: {
    component: ProductGridSection,
    schema: ProductGridSectionSchema,
  },
} as const;
```

Renderer:

```tsx
function renderSection(section: UnknownRuntimeSection) {
  const entry = sectionRegistry[section.type];

  if (!entry) {
    return <UnsupportedSection sectionType={section.type} />;
  }

  const result = entry.schema.safeParse(section.props);

  if (!result.success) {
    return (
      <InvalidSection
        sectionId={section.id}
        errors={result.error.issues}
      />
    );
  }

  const Component = entry.component;

  return <Component {...result.data} />;
}
```

Flow lúc này là:

```text
Receive configuration
        ↓
Resolve registry entry
        ↓
Validate props with matching schema
        ↓
Render approved component
```

---

## Registry metadata

Khi hệ thống mở rộng, registry có thể cần thêm metadata:

```ts
const sectionRegistry = {
  hero: {
    component: HeroSection,
    schema: HeroSectionSchema,
    status: "stable",
    supportedVersions: [1, 2],
    editor: {
      label: "Hero",
      category: "Marketing",
      icon: "layout-hero",
    },
    fallback: HeroFallback,
  },
};
```

Metadata có thể phục vụ:

* Runtime renderer.
* CMS editor.
* Component picker.
* Schema validation.
* Documentation generation.
* Version compatibility.
* Deprecation warning.
* Feature availability.
* Fallback rendering.

Registry lúc này trở thành một source of truth cho toàn bộ runtime capability.

---

## Versioning và deprecation

Một component identifier không nên bị thay đổi tùy tiện sau khi đã được publish.

Ví dụ, thay vì xóa `hero` ngay lập tức:

```ts
const sectionRegistry = {
  hero: {
    status: "deprecated",
    replacement: "heroV2",
    component: LegacyHeroSection,
  },

  heroV2: {
    status: "stable",
    component: HeroSectionV2,
  },
};
```

Điều này cho phép:

* Config cũ tiếp tục hoạt động.
* CMS ngừng tạo config mới với component deprecated.
* Hệ thống migrate dần sang version mới.
* Telemetry theo dõi lượng config còn dùng version cũ.

---

## Failure isolation

Một section lỗi không nên làm crash toàn bộ trang.

```tsx
<SectionErrorBoundary
  sectionId={section.id}
  sectionType={section.type}
>
  <Component {...props} />
</SectionErrorBoundary>
```

Registry có thể định nghĩa fallback riêng:

```ts
hero: {
  component: HeroSection,
  fallback: HeroFallback,
}
```

Điều này đặc biệt hữu ích khi mỗi component có failure behavior khác nhau.

---

## Những anti-pattern phổ biến

### Dynamic import từ giá trị tùy ý

```ts
import(config.componentPath);
```

Configuration không nên quyết định arbitrary module path.

### Registry chỉ tồn tại trong renderer

Nếu CMS, validator và documentation dùng ba danh sách component khác nhau, contract sẽ sớm mất đồng bộ.

### Không quản lý deprecation

Xóa một registry entry có thể làm hỏng toàn bộ config cũ.

### Dùng `any` cho toàn bộ props

Runtime vẫn hoạt động, nhưng mất contract giữa type, schema và component.

### Registry chứa business logic

Registry nên mô tả capability và mapping, không nên trở thành nơi chứa workflow hoặc logic nghiệp vụ phức tạp.

---

## Trade-offs

### Lợi ích

* Tách configuration khỏi implementation detail.
* Kiểm soát component được runtime phép sử dụng.
* Tăng khả năng refactor frontend.
* Hỗ trợ validation và type safety.
* Hỗ trợ CMS component picker.
* Có thể quản lý version và deprecation tập trung.
* Tạo source of truth cho runtime capability.

### Chi phí

* Registry có thể trở thành module lớn.
* Type inference phức tạp khi component rất khác nhau.
* Cần quy trình quản lý identifier ổn định.
* Metadata có nguy cơ phình to.
* Cần tránh coupling toàn bộ hệ thống vào một file duy nhất.

Khi registry mở rộng, có thể chia theo domain:

```text
registries/
├── marketing.registry.ts
├── commerce.registry.ts
├── content.registry.ts
└── index.ts
```

Nhưng public identifier vẫn phải duy trì uniqueness và backward compatibility.

---

## Kết luận

Một Component Registry production-ready không chỉ trả lời:

> Với `type` này, cần render component nào?

Nó còn phải xác định:

* Runtime được phép sử dụng capability nào?
* Props nào được chấp nhận?
* Schema version nào được hỗ trợ?
* Component nào đã deprecated?
* Fallback nào được sử dụng khi có lỗi?
* CMS có thể cung cấp option nào cho business?

Nếu thiếu boundary này, Runtime UI không loại bỏ coupling.

Nó chỉ di chuyển coupling từ source code sang configuration.

Vì vậy:

> Component Registry là public API và safety boundary giữa runtime configuration với frontend implementation.

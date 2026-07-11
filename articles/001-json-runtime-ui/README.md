# JSON Runtime UI trong dự án Mini App: Giảm những lần release không cần thiết

## Bối cảnh

Trong một dự án outsourcing chạy trên Zalo Mini App, chúng tôi gặp một constraint đáng chú ý:

Một thay đổi trong source code không kết thúc sau khi developer hoàn thành công việc.

Nó có thể tiếp tục đi qua một delivery cycle gồm:

```text
Development
    ↓
Code review
    ↓
Build
    ↓
Submit
    ↓
Platform review
    ↓
Release
```

Trong khi đó, khách hàng thường xuyên có những yêu cầu liên quan đến campaign:

* Thay banner.
* Thay nội dung CTA.
* Đổi thứ tự section.
* Bật hoặc tắt một khu vực.
* Chuyển layout variant.
* Điều chỉnh nội dung theo từng giai đoạn.

Một yêu cầu có thể rất nhỏ về mặt implementation nhưng tạo ra lead time lớn hơn đáng kể khi đặt trong toàn bộ quy trình delivery.

Vấn đề cần giải quyết vì thế không chỉ là:

> Làm thế nào để developer code nhanh hơn?

Mà là:

> Làm thế nào để giảm số lượng thay đổi bắt buộc phải trở thành source-code change?

---

## Phân loại thay đổi

Trước khi lựa chọn giải pháp, chúng tôi chia các thay đổi thành hai nhóm.

### Behavior

Đây là những phần nên tiếp tục được kiểm soát bằng source code:

* Authentication.
* Authorization.
* Payment flow.
* Validation.
* Navigation behavior.
* API orchestration.
* State transition.
* Complex interaction.
* Business rules.

### Configuration

Đây là những phần có thể được cân nhắc điều khiển ở runtime:

* Nội dung.
* Hình ảnh.
* Thứ tự section.
* Visibility.
* Component variant.
* Campaign identifier.
* Layout option.
* Theme token trong phạm vi cho phép.

Ranh giới này là nền tảng của toàn bộ quyết định kiến trúc.

JSON Runtime UI không nhằm chuyển toàn bộ ứng dụng thành dữ liệu. Nó chỉ di chuyển những thay đổi có bản chất là configuration ra khỏi release cycle.

---

## Cách tiếp cận hard-coded

Một màn hình truyền thống có thể được định nghĩa trực tiếp:

```tsx
export function HomePage() {
  return (
    <>
      <HeroSection />
      <CampaignSection />
      <ProductGrid />
      <CallToAction />
    </>
  );
}
```

Cấu trúc trên đơn giản, dễ đọc và type-safe.

Tuy nhiên, nếu thứ tự hoặc variant của các section phải thay đổi thường xuyên, mỗi yêu cầu đều có khả năng dẫn đến một commit và một lần release mới.

---

## Chuyển sang Runtime Configuration

Frontend giữ một tập hợp component đã được kiểm soát:

```ts
const sectionRegistry = {
  hero: HeroSection,
  campaign: CampaignSection,
  productGrid: ProductGridSection,
  callToAction: CallToActionSection,
} as const;
```

Cấu trúc màn hình được mô tả bằng JSON:

```json
{
  "schemaVersion": 1,
  "page": "home",
  "sections": [
    {
      "id": "main-hero",
      "type": "hero",
      "variant": "campaign",
      "visible": true,
      "props": {
        "title": "Summer Campaign",
        "image": "/campaigns/summer/hero.webp"
      }
    },
    {
      "id": "featured-campaign",
      "type": "campaign",
      "visible": true,
      "props": {
        "campaignId": "summer-2026"
      }
    }
  ]
}
```

Renderer ánh xạ configuration sang component:

```tsx
function RuntimePage({ sections }: RuntimePageProps) {
  return sections.map((section) => {
    if (!section.visible) {
      return null;
    }

    const Component = sectionRegistry[section.type];

    if (!Component) {
      return <UnsupportedSection key={section.id} />;
    }

    return (
      <SectionErrorBoundary key={section.id}>
        <Component
          variant={section.variant}
          {...section.props}
        />
      </SectionErrorBoundary>
    );
  });
}
```

Backend hoặc CMS không gửi JavaScript và không quyết định implementation cụ thể.

Nó chỉ lựa chọn những component và option đã được frontend cho phép.

---

## Component Registry là một whitelist

Runtime không nên được phép gọi arbitrary component.

Configuration không nên biết đến đường dẫn nội bộ như:

```text
src/features/campaign/components/HeroSectionV4.tsx
```

Nó chỉ nên biết public identifier:

```text
hero
campaign
productGrid
callToAction
```

Registry tạo ra abstraction boundary giữa runtime configuration và implementation của frontend.

Frontend có thể refactor component bên trong mà không bắt buộc phải thay đổi toàn bộ configuration đã publish.

---

## Validation là bắt buộc

TypeScript không thể tự bảo vệ dữ liệu nhận từ network.

Configuration phải được validate ở runtime trước khi render.

Ví dụ với Zod:

```ts
import { z } from "zod";

const HeroSectionSchema = z.object({
  id: z.string(),
  type: z.literal("hero"),
  variant: z.enum(["default", "campaign", "minimal"]),
  visible: z.boolean(),
  props: z.object({
    title: z.string(),
    image: z.string().optional(),
  }),
});

const RuntimePageSchema = z.object({
  schemaVersion: z.literal(1),
  page: z.string(),
  sections: z.array(
    z.discriminatedUnion("type", [
      HeroSectionSchema,
      CampaignSectionSchema,
      ProductGridSectionSchema,
    ])
  ),
});
```

Validation nên diễn ra ở nhiều thời điểm:

```text
CMS editing
    ↓
Pre-publish validation
    ↓
API response validation
    ↓
Client-side validation
```

Không nên đợi đến thiết bị người dùng mới phát hiện configuration sai.

---

## Publish flow

Khả năng thay đổi runtime nhanh hơn cũng làm tăng rủi ro phá production nhanh hơn.

Do đó, configuration cần một publish lifecycle rõ ràng:

```text
Draft
   ↓
Validate
   ↓
Preview
   ↓
Approve
   ↓
Publish
   ↓
Active version
```

Mỗi lần publish nên tạo một immutable version:

```text
home:v12
home:v13
home:v14
```

Nếu `v14` gây lỗi, hệ thống có thể rollback về `v13` mà không cần phát hành lại ứng dụng.

---

## Last-known-good configuration

Client nên có phương án khi không thể tải configuration mới:

```text
Fetch active configuration
           ↓
       Successful?
       /         \
     Yes          No
      ↓            ↓
Validate       Load cached
      ↓        last-known-good
Render
```

Fallback có thể bao gồm:

* Configuration được đóng gói sẵn trong app.
* Version gần nhất đã render thành công.
* Minimal safe page.
* Ẩn section lỗi thay vì crash toàn bộ màn hình.

---

## Những gì không nên đưa vào JSON

JSON Runtime UI bắt đầu trở nên nguy hiểm khi configuration được dùng như một programming language.

Các dấu hiệu cần dừng lại:

* Condition lồng nhiều tầng.
* Expression tùy ý.
* Loop.
* State mutation.
* Side effect.
* Network orchestration.
* Business rule quan trọng.
* Quyền truy cập được quyết định từ client config.

Nguyên tắc của chúng tôi:

> JSON mô tả UI cần hiển thị, nhưng không sở hữu business behavior cốt lõi.

Nếu một requirement cần nhiều branching và state transition, nó nên được triển khai thành một component hoặc feature mới trong source code.

---

## Trade-offs

### Lợi ích

* Giảm một số lần release không cần thiết.
* Phản hồi campaign nhanh hơn.
* Giảm dependency giữa Business và Developer.
* Tái sử dụng component giữa nhiều màn hình.
* Cho phép preview và rollback configuration.
* Một frontend renderer có thể phục vụ nhiều cấu trúc trang.

### Chi phí

* Cần xây dựng schema và registry.
* Debugging phụ thuộc vào config version.
* Phải quản lý backward compatibility.
* CMS cần guardrail tốt.
* Cần telemetry và error reporting theo từng section.
* Phải ngăn configuration phát triển thành logic không kiểm soát.

JSON Runtime UI chỉ có giá trị khi chi phí xây dựng hệ thống thấp hơn chi phí của những lần thay đổi lặp lại trong tương lai.

---

## Kết luận

Phần đơn giản nhất của Runtime UI là render một component từ JSON.

Phần khó hơn là thiết kế:

* Ranh giới giữa behavior và configuration.
* Schema contract.
* Component registry.
* Versioning.
* Validation.
* Rollback.
* Failure handling.
* Observability.

Trong dự án này, JSON Runtime UI không được lựa chọn vì đây là một pattern mới hoặc thú vị.

Nó xuất phát từ một constraint rất cụ thể của delivery process.

Mục tiêu cuối cùng không phải là loại bỏ source code hay loại bỏ release.

Mục tiêu là:

> Di chuyển những thay đổi không cần thiết phải là source-code change ra khỏi release cycle.

Đó cũng là nguyên tắc xuyên suốt của **weTwenties Engineering Playbook**:

> Thiết kế hệ thống không chỉ để giải quyết yêu cầu hiện tại, mà còn để giảm chi phí của những thay đổi tương tự trong tương lai.

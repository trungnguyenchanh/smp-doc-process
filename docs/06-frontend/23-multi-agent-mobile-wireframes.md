# Mobile UX Wireframes · Multi-Agent Step Assignment (v3.5+)

> **Status**: DRAFT for Mobile UX review  
> **Date**: 2026-05-29  
> **Audience**: Mobile Lead + Mobile Dev + Founder review  
> **Format**: ASCII wireframes + interaction flows  
> **Reference**: [Doc 09 Epic 7 User Stories](../05-ba/09-user-stories.md), [Doc 15 Section Q Rules](../05-ba/15-business-rules.md)

---

## Screen 1 · Step detail (Lead view) · Step chưa có helpers

```text
┌─────────────────────────────────────────┐
│ ←  Step 3 · Lắp đặt              ⓘ     │ ← header với info icon
├─────────────────────────────────────────┤
│                                          │
│ Đơn #ORD-456                            │
│ KH: Anh Nguyễn Văn A                    │
│ ⏰ Bắt đầu: 10:30 hôm nay               │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ 💰 Doanh thu step này                   │
│ 300,000 VND (30% của đơn 1,000,000)    │
│                                          │
│ Bạn nhận: 300,000 VND (100%)            │
│ ⭐ Bạn là Lead                          │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ 👥 Team (1 người)                       │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 🌟 Bạn                              │ │
│ │    Lead · 100% · 300,000 VND        │ │
│ │    Đang làm                          │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ ➕ Mời thợ phụ                       │ │ ← CTA button
│ │    (helper hoặc thợ chuyên môn)     │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ [ ✅ Hoàn thành step ]                  │ ← primary action
│                                          │
└─────────────────────────────────────────┘
```

**Interaction**:
- Tap "Mời thợ phụ" → Screen 2 (Invite helper modal)
- Tap "Hoàn thành step" → confirmation modal → trigger earnings calc

---

## Screen 2 · Invite helper modal (Lead view)

```text
┌─────────────────────────────────────────┐
│ ✕  Mời thợ phụ                          │
├─────────────────────────────────────────┤
│                                          │
│ Loại thợ                                 │
│ ┌─────────────────────────────────────┐ │
│ │ ○ Thợ phụ (helper)                  │ │ ← radio
│ │ ● Thợ chuyên môn (specialist) ✓     │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ Chuyên môn (nếu specialist)             │
│ ┌─────────────────────────────────────┐ │
│ │ ⚡ Thợ điện (electrician)      ✓    │ │ ← dropdown selected
│ └─────────────────────────────────────┘ │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Thợ gần đây · sẵn sàng nhận             │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 👤 Anh Trần B                  ⭐ 4.8│ │
│ │    Cách 2.3km · L3 electrician     │ │
│ │    [ Mời ]                          │ │ ← invite button
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 👤 Anh Lê C                    ⭐ 4.5│ │
│ │    Cách 3.1km · L2 electrician     │ │
│ │    [ Mời ]                          │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 👤 Anh Phạm D                  ⭐ 4.9│ │
│ │    Cách 5.0km · L4 electrician     │ │
│ │    [ Mời ]                          │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ [ Xem thêm thợ ↓ ]                      │
│                                          │
└─────────────────────────────────────────┘
```

**Interaction**:
- Tap "Mời" trên thợ B → Screen 3 (Set split ratio)
- Specialist dropdown options: electrician, plumber, network, hvac
- Filter sort: distance ASC default, có thể switch sang rating DESC

---

## Screen 3 · Set split ratio modal (after invite)

```text
┌─────────────────────────────────────────┐
│ ← Mời thợ Trần B · Set tỉ lệ            │
├─────────────────────────────────────────┤
│                                          │
│ Chia tiền cho team (tổng = 100%)         │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 🌟 Bạn (Lead)                       │ │
│ │    [  60%  ]  ←──→  180,000 VND     │ │ ← slider
│ │                                      │ │
│ ├─────────────────────────────────────┤ │
│ │ ⚡ Anh Trần B (Electrician)         │ │
│ │    [  40%  ]  ←──→  120,000 VND     │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ✓ Tổng: 100% (300,000 VND)             │ ← validation OK
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Mặc định service "Sửa máy lạnh":        │ ← hint
│ Lead 50% · Electrician 50%              │
│ [ Reset về mặc định ]                   │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Lý do tùy chỉnh (optional)              │
│ ┌─────────────────────────────────────┐ │
│ │ Tôi làm phần khó hơn _________      │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ [ Hủy ]            [ ✉️ Gửi lời mời ]  │
│                                          │
└─────────────────────────────────────────┘
```

**Validation states**:
- Tổng < 100% → red warning "Còn thiếu X%", Send button disabled
- Tổng = 100% → green check, Send enabled
- Tổng > 100% → red warning "Thừa X%", Send disabled

**After Send**:
- Helper nhận push notification
- Lead screen quay về Screen 1 với status "Đang chờ Trần B accept"

---

## Screen 4 · Step detail (Lead view) · Sau khi helper accept

```text
┌─────────────────────────────────────────┐
│ ←  Step 3 · Lắp đặt              ⓘ     │
├─────────────────────────────────────────┤
│                                          │
│ 💰 Doanh thu step này                   │
│ 300,000 VND                              │
│                                          │
│ Bạn nhận: 180,000 VND (60%)             │
│ ⭐ Bạn là Lead                          │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ 👥 Team (2 người)                  [ ⚙️ ]│ ← settings = adjust splits
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 🌟 Bạn                              │ │
│ │    Lead · 60% · 180,000 VND         │ │
│ │    Đang làm                          │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ ⚡ Anh Trần B (Electrician)         │ │
│ │    Specialist · 40% · 120,000 VND   │ │
│ │    ✅ Đã accept                     │ │
│ │                                      │ │
│ │    [ Xóa khỏi step ] [ Đổi tỉ lệ ] │ │ ← lead actions
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ ➕ Mời thêm thợ phụ                  │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ [ ✅ Hoàn thành step ]                  │
│                                          │
└─────────────────────────────────────────┘
```

---

## Screen 5 · Step detail (Helper/Specialist view)

```text
┌─────────────────────────────────────────┐
│ ←  Step 3 · Lắp đặt              ⓘ     │
├─────────────────────────────────────────┤
│                                          │
│ Đơn #ORD-456                            │
│ KH: Anh Nguyễn Văn A                    │
│ ⏰ Bắt đầu: 10:30 hôm nay               │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ 💰 Bạn nhận                              │
│ 120,000 VND (40% của step này)          │
│                                          │
│ Vai trò: Specialist · Electrician        │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ 👤 Lead của step                         │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 🌟 Anh Trần A                       │ │
│ │    Lead · L4                        │ │
│ │    📞 Gọi  💬 Chat                  │ │ ← contact actions
│ └─────────────────────────────────────┘ │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ ⓘ Tỉ lệ chia tiền do Lead quyết định    │ ← privacy note
│   Bạn không xem được tỉ lệ của          │
│   các thợ khác.                          │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Trạng thái: ✅ Đã accept                │
│ [ Bắt đầu làm việc ]                    │
│                                          │
└─────────────────────────────────────────┘
```

**Privacy compliance** (BR-MA-010): helper KHÔNG thấy splits của lead và các thợ khác. Chỉ thấy split của mình.

---

## Screen 6 · Invitation notification (Helper view)

```text
┌─────────────────────────────────────────┐
│  🔔 Lời mời mới · 30 phút còn lại       │ ← timer
├─────────────────────────────────────────┤
│                                          │
│ Anh Trần A mời bạn vào step             │
│ "Lắp đặt" của đơn #ORD-456              │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 📍 Quận 7, TP.HCM (cách 2.3km)     │ │
│ │ ⏰ Bắt đầu: 10:30 hôm nay           │ │
│ │ ⏱️ Dự kiến: 1h                      │ │
│ │ 🛠️ Vai trò: Electrician            │ │
│ │                                      │ │
│ │ 💰 Bạn nhận: 120,000 VND (40%)     │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ Service: Lắp đặt máy lạnh                │
│ Step 3/5 của đơn                         │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ [ ❌ Từ chối ]    [ ✅ Nhận lời mời ]   │
│                                          │
└─────────────────────────────────────────┘
```

**Timer behavior**: 30 phút auto-reject nếu không action (BR-MA-007). Countdown visible.

**On reject**: prompt reason picker:
```text
┌─────────────────────────────────────────┐
│ Lý do từ chối                            │
├─────────────────────────────────────────┤
│ ○ Quá xa                                 │
│ ○ Bận việc khác                          │
│ ○ Sai chuyên môn                         │
│ ○ Cá nhân                                │
│ ○ Khác (ghi rõ)                          │
│                                          │
│ [ Hủy ]                  [ Gửi ]        │
└─────────────────────────────────────────┘
```

---

## Screen 7 · Earnings breakdown (per step view)

```text
┌─────────────────────────────────────────┐
│ ←  Thu nhập tháng 6/2026                │
├─────────────────────────────────────────┤
│                                          │
│ Tổng: 8,500,000 VND                     │
│ ─────────────────────────────           │
│                                          │
│ 📊 Theo vai trò                         │
│ ▓▓▓▓▓▓▓▓▓▓▓░░  Lead       6,200,000    │
│ ▓▓▓░░░░░░░░░░  Helper       1,500,000   │
│ ▓▓░░░░░░░░░░░  Specialist     800,000   │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Lịch sử đơn                              │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 📅 15/06 · ORD-456                  │ │
│ │ Lắp đặt máy lạnh                    │ │
│ │ 5 steps · Lead                       │ │
│ │ Nhận: 680,000 VND  ›                 │ │ ← tap to expand
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 📅 14/06 · ORD-455                  │ │
│ │ Lắp camera                          │ │
│ │ 4 steps · Specialist (Electrician)  │ │
│ │ Nhận: 240,000 VND  ›                 │ │
│ └─────────────────────────────────────┘ │
│                                          │
└─────────────────────────────────────────┘
```

**Tap expand ORD-456** → Screen 8:

---

## Screen 8 · Order earnings detail (per step breakdown)

```text
┌─────────────────────────────────────────┐
│ ←  ORD-456 · Earnings detail            │
├─────────────────────────────────────────┤
│                                          │
│ Lắp đặt máy lạnh                         │
│ Hoàn thành: 15/06 17:30                 │
│ Doanh thu đơn: 1,000,000 VND            │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Tổng bạn nhận: 680,000 VND              │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Chi tiết theo step                       │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ Step 1 · Khảo sát                   │ │
│ │ Vai trò: Lead · 100%                │ │
│ │ Step revenue: 80,000 VND            │ │
│ │ Bạn nhận: 80,000 VND                │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ Step 2 · Đục tường + đi ống         │ │
│ │ Vai trò: Lead · 50%                 │ │
│ │ Step revenue: 250,000 VND           │ │
│ │ Bạn nhận: 125,000 VND               │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ Step 3 · Lắp dàn nóng               │ │
│ │ Vai trò: Lead · 50%                 │ │
│ │ Step revenue: 250,000 VND           │ │
│ │ Bạn nhận: 125,000 VND               │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ Step 4 · Lắp dàn lạnh + đấu điện    │ │
│ │ Vai trò: Lead · 40%                 │ │
│ │ Step revenue: 270,000 VND           │ │
│ │ Bạn nhận: 108,000 VND               │ │
│ │                                      │ │
│ │ 👥 Team breakdown (lead-only view)  │ │
│ │ ├ 🌟 Bạn (Lead): 40%               │ │
│ │ ├ ⚡ Trần B (Electrician): 40%     │ │
│ │ └ 👤 Lê C (Helper): 20%            │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ Step 5 · Test + nghiệm thu          │ │
│ │ Vai trò: Lead · 60%                 │ │
│ │ Step revenue: 150,000 VND           │ │
│ │ Bạn nhận: 90,000 VND                │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ─────────────────────────────           │
│                                          │
│ Tổng: 80k + 125k + 125k + 108k + 90k    │
│     = 680,000 VND ✓                     │
│                                          │
└─────────────────────────────────────────┘
```

**Team breakdown visibility** (BR-MA-010):
- Lead view: full breakdown (transparency for team management)
- Helper/Specialist view: chỉ thấy step + split của mình, không thấy split của thợ khác

---

## Interaction flows

### Flow 1: Lead invites helper successfully

```text
[Step detail · 1 lead only]
    ↓ tap "Mời thợ phụ"
[Invite modal: chọn loại + specialty]
    ↓ tap "Mời" trên thợ
[Set split modal]
    ↓ adjust sliders to 60/40
    ↓ tap "Gửi lời mời"
[Back to Step detail · status "Đang chờ accept"]
    ⏰ wait...
[Push notification arrives: Helper accepted]
    ↓ refresh
[Step detail · 2 agents · status all OK]
```

### Flow 2: Helper rejects invitation

```text
[Push notification on helper phone]
    ↓ tap notification
[Invitation screen with 30-min timer]
    ↓ tap "Từ chối"
[Reason picker]
    ↓ select "Quá xa" + tap "Gửi"
[Back to home screen]

Lead side:
[Push notification: Helper rejected]
    ↓ tap notification
[Step detail · status "Trần B từ chối · lý do: Quá xa"]
[Option: invite another helper OR proceed solo]
```

### Flow 3: Lead adjusts splits mid-step

```text
[Step detail · 2 agents accepted, in_progress]
    ↓ tap settings ⚙️ → "Đổi tỉ lệ chia tiền"
[Adjust splits modal · 60/40 → 65/35]
    ↓ validation: SUM still = 100% ✓
    ↓ tap "Lưu thay đổi"
[Confirmation: "Đã cập nhật. Helper sẽ nhận thông báo"]

Helper side:
[Push: Lead đã cập nhật tỉ lệ chia tiền của bạn]
    ↓ tap to view
[Step detail · new split shown: 35% · 105,000 VND]
```

---

## Design notes

### Currency formatting
- Always show VND with thousand separator (locale `vi-VN`): `120,000 VND` not `120000`
- Per Doc 04 §1.15 use `pkg/money` formatter

### Percentage display
- Internal storage: bps (10000 = 100%)
- UI display: percent (0.01% precision OK)
- Slider step: 5% (500 bps) for usability
- Manual input allowed: 1% (100 bps) increment

### Lead role indicator
- Star icon ⭐ throughout to distinguish lead
- Color: indigo `#4f46e5` (theme primary)

### Specialty icons
- ⚡ electrician
- 🔧 plumber
- 🌐 network
- 🛠️ generic helper

### Privacy banner
- Helper view always shows: "ⓘ Tỉ lệ chia tiền do Lead quyết định. Bạn không xem được tỉ lệ của các thợ khác."

### Error states
- Split sum ≠ 100% → red banner + Send disabled
- Helper offline → grey out invite button + tooltip "Thợ đang offline"
- Lead trying to remove agent with status='completed' → modal "Không thể xóa thợ đã hoàn thành"

### Loading states
- Invite list: skeleton 3 cards
- Earnings: shimmer effect

### Accessibility
- Color contrast WCAG AA minimum
- Screen reader: "Tổng tỉ lệ là 100 phần trăm" announce when split changes
- Tap targets ≥ 44pt iOS / 48dp Android

---

## Implementation priority (cho Sprint 1 mockup review)

| Priority | Screen | Story |
|---|---|---|
| P0 | Screen 1 (Step detail Lead) | US-7.1 |
| P0 | Screen 2 (Invite helper modal) | US-7.1 |
| P0 | Screen 3 (Set split ratio) | US-7.1 |
| P0 | Screen 6 (Invitation push) | US-7.1 |
| P1 | Screen 4 (Step detail after accept) | US-7.1 |
| P1 | Screen 5 (Helper view) | US-7.1 |
| P1 | Earnings screen extensions (Screen 7, 8) | US-7.3 |
| P2 | Lead adjust splits (settings flow) | US-7.2 |

P0 = ship with Sprint 3-4. P1 = ship with Sprint 4. P2 = ship Sprint 5+.

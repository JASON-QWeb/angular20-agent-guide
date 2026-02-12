# Angular 20+ 前端开发指南 (Agent 规则版)

本指南旨在为 AI Agent 提供最新的 Angular 20+ 开发规范、语法参考及 Ant Design (NG-ZORRO) 集成最佳实践。

---

## 1. 核心架构原则 (Core Philosophy)

### 1.1 全面信号化 (Signal-Based)
Angular 20 已经进入“信号时代”。**禁止**在组件中使用传统的成员变量进行状态管理（除非是常量），**严禁**过度依赖 `Observable` 处理组件内部状态。
- **状态存储**：使用 `signal()`。
- **派生状态**：使用 `computed()`。
- **副作用**：使用 `effect()`。

### 1.2 无区运行 (Zoneless)
Angular 20 默认推荐无区架构（Zoneless）。
- **配置**：在 `app.config.ts` 中使用 `provideExperimentalZonelessChangeDetection()`。
- **后果**：不再依赖 `zone.js` 触发变更检测。只有信号更新、DOM 事件、或手动调用 `markForCheck()` 才会触发渲染。

### 1.3 独立组件 (Standalone)
**禁止**使用 `NgModule`。所有组件、指令、管道必须声明为 `standalone: true`。

---

## 2. 现代语法参考 (Modern Syntax)

### 2.1 组件定义 (Component)
```typescript
@Component({
  selector: 'app-user-profile',
  standalone: true,
  imports: [CommonModule, NzAvatarModule, NzButtonModule], // 按需引入 NG-ZORRO 模块
  template: `
    <div class="profile-container">
      @if (user(); as u) {
        <nz-avatar [nzText]="u.name"></nz-avatar>
        @let greeting = '你好, ' + u.name;
        <h1>{{ greeting }}</h1>
      } @else {
        <nz-skeleton-element nzType="avatar"></nz-skeleton-element>
      }
    </div>
  `
})
export class UserProfileComponent {
  // Input 信号
  user = input.required<User>();
  
  // Model 信号 (双向绑定)
  status = model<'online' | 'offline'>('online');
  
  // ViewChild 信号
  submitBtn = viewChild<NzButtonComponent>('submitBtn');
}
```

### 2.2 模板控制流 (Control Flow)
**禁止**使用 `*ngIf`, `*ngFor`。必须使用 `@` 语法。
- **Conditional**: `@if (condition) { ... } @else if { ... } @else { ... }`
- **Loop**: `@for (item of list(); track item.id) { ... } @empty { 列表为空 }`
- **Switch**: `@switch (state()) { @case ('A') { ... } @default { ... } }`
- **Local Variable**: `@let name = user().firstName + ' ' + user().lastName;`

### 2.3 数据获取 (Data Fetching)
推荐使用 `rxResource` 或 `http` 与 `toSignal` 结合。
```typescript
import { rxResource } from '@angular/core/rxjs-interop';

export class DataComponent {
  userId = signal(1);
  
  // 自动响应 userId 变化并重新获取数据
  userResource = rxResource({
    request: () => ({ id: this.userId() }),
    loader: ({ request }) => this.http.get<User>(`/api/users/${request.id}`)
  });
  
  // 使用方式：userResource.value()
}
```

---

## 3. NG-ZORRO (Ant Design) 集成规范

### 3.1 属性绑定
由于组件属性现在多为信号，在模板中绑定 NG-ZORRO 属性时需注意加括号：
```html
<nz-table [nzData]="items()" [nzLoading]="loading()">
  <thead>
    <tr>
      <th>名称</th>
    </tr>
  </thead>
  <tbody>
    @for (data of items(); track data.id) {
      <tr>
        <td>{{ data.name }}</td>
      </tr>
    }
  </tbody>
</nz-table>
```

### 3.2 表单处理 (Reactive Forms)
即便在信号时代，`ReactiveFormsModule` 依然是处理复杂表单的首选。但可以用信号包装表单状态。
```typescript
form = new FormGroup({
  username: new FormControl('', Validators.required)
});

// 将表单值转为信号
formValue = toSignal(this.form.valueChanges, { initialValue: this.form.value });
```

---

## 4. 常见错误与避坑指南 (Common Pitfalls)

1.  **忘记调用信号**：在模板或 `computed` 中忘记加 `()`。
    - ❌ `{{ user.name }}`
    - ✅ `{{ user().name }}`
2.  **在 effect 中修改信号**：会导致无限循环。如需修改，确保逻辑正确或使用 `allowSignalWrites: true`（慎用）。
3.  **过度使用 toObservable**：除非需要 RxJS 的高级操作符（如 `switchMap`, `debounceTime`），否则应留在信号域。
4.  **Zoneless 渲染失效**：在第三方库（如非 Angular 原生的图表库）异步回调中修改普通变量。
    - **修复**：必须通过信号修改，或使用 `ChangeDetectorRef.markForCheck()`。
5.  **内存泄漏**：在 `effect()` 中使用了外部资源。
    - **修复**：在 `onCleanup` 回调中清理。

---

## 6. 推荐项目结构 (Recommended Structure)
```
src/
├── app/
│   ├── core/           # 全局单例服务, 拦截器, Guard
│   ├── shared/         # 公用组件, 指令, 管道 (都是 Standalone)
│   ├── features/       # 业务特性模块
│   │   └── user/
│   │       ├── user-list.component.ts
│   │       ├── user-detail.component.ts
│   │       └── user.service.ts
│   ├── app.config.ts   # 替代 app.module.ts 的全局配置
│   └── app.routes.ts   # 路由定义
```

## 7. 高级特性 (Advanced Features)

### 7.1 水合与服务端渲染 (SSR & Hydration)
Angular 20 默认开启水合。
- **配置**：`provideClientHydration(withIncrementalHydration())`。
- **注意**：避免在 `constructor` 中直接操作 DOM，应使用 `afterRender` 或 `afterNextRender` 生命周期。

### 7.2 局部变量 `@let`
`@let` 可以用于在模板中解构信号值或存储复杂的布尔逻辑，显著提高可读性。
```html
@let loading = userResource.isLoading();
@let error = userResource.error();

@if (loading) {
  <nz-spin></nz-spin>
} @else if (error) {
  <nz-alert nzType="error" [nzMessage]="error"></nz-alert>
}
```

## 8. 自动化测试建议
- **单元测试**：使用 `ComponentFixture.setComponentInputs` 测试信号 Input。
- **组件测试**：利用 `TestBed.runInInjectionContext` 测试自定义信号操作符。


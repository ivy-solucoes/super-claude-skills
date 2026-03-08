---
name: frappe-ui
description: >
  Desenvolvimento frontend com Frappe UI — biblioteca oficial de componentes Vue 3 para apps
  Frappe/ERPNext. Use esta skill SEMPRE que o usuário mencionar frappe-ui, componentes Vue no
  contexto Frappe, frontend de app Frappe, ou qualquer um dos seguintes: Button, Dialog, ListView,
  Combobox, DatePicker, TextInput, Textarea, Select, Tabs, Sidebar, Dropdown, Badge, Avatar,
  Alert, Breadcrumbs, Checkbox, Switch, FileUploader, Tooltip, Progress, Popover, Slider, Rating,
  MultiSelect, Password, TextEditor, TimePicker, MonthPicker, Calendar, Charts, Tree, ErrorMessage.
  Também use quando o usuário trabalhar com createResource, createListResource, createDocumentResource,
  frappeRequest, data fetching no Frappe, ou quiser configurar um projeto Vue com frappe-ui-starter.
---

# Frappe UI — Referência Completa

Biblioteca de componentes Vue 3 oficial do ecossistema Frappe. Versão atual: **v0.1.265**
Documentação: https://ui.frappe.io/docs

---

## 1. Setup

### Novo projeto do zero
```sh
bench new-app meuapp
cd apps/meuapp
npx degit netchampfaris/frappe-ui-starter frontend
```

### Desabilitar CSRF no dev (obrigatório para evitar erros com o Vite)
```sh
bench --site meusite.test set-config ignore_csrf 1
```
Em produção, o `csrf_token` é injetado automaticamente no `index.html`.

### Iniciar dev server
```sh
cd frontend
yarn && yarn dev
# Acessa: http://meusite.test:8080
```
O Vite roda na porta `8080` e faz proxy para o Frappe (porta 8000).
Para mudar a base URL de produção, edite `src/router.js` → `createWebHistory(base)`.

### Configurar frappeRequest (obrigatório para integração com backend Frappe)
Em `main.js`:
```js
import { setConfig, frappeRequest } from 'frappe-ui'
setConfig('resourceFetcher', frappeRequest)
```
Com isso, as URLs podem ser abreviadas (sem `/api/method/`) e a resposta é extraída do campo `message`.

### Registrar resourcesPlugin (para Options API)
```js
import { resourcesPlugin } from 'frappe-ui'
app.use(resourcesPlugin)
```

---

## 2. Data Fetching

### 2.1 createResource — requisição genérica

```js
import { createResource } from 'frappe-ui'

let todos = createResource({
  url: 'frappe.client.get_list',       // URL ou método Frappe
  method: 'GET',                        // padrão: POST
  params: { doctype: 'ToDo' },
  auto: true,                           // busca automaticamente ao criar
  cache: 'todos',                       // cache em memória e IndexedDB
  debounce: 500,
  initialData: [],
  makeParams() { return { doctype: 'ToDo' } },
  validate(params) {
    if (!params.doctype) return 'doctype is required'
  },
  transform(data) {
    return data.map(d => ({ ...d, open: false }))
  },
  onSuccess(data) { console.log(data) },
  onError(error) { console.error(error) },
})
```

**API do resource:**
```js
todos.data           // dados retornados
todos.loading        // true enquanto carrega
todos.error          // erro da requisição ou de validate()
todos.promise        // promise da requisição (await)
todos.fetched        // true após primeira busca
todos.previousData   // dados anteriores ao reload

todos.fetch()        // dispara requisição
todos.reload()       // alias de fetch
todos.submit({ id: 2 }) // fetch com parâmetros
todos.reset()        // reseta para estado inicial
todos.update({ url: '/api/users', params: { id: 2 } })
todos.setData({ id: 1, title: 'test' })
todos.setData(data => data.filter(d => d.open))
```

---

### 2.2 createListResource — lista de documentos Frappe

```js
import { createListResource } from 'frappe-ui'

let tenants = createListResource({
  doctype: 'Tenant',
  fields: ['name', 'tenant_type', 'email', 'status'],
  filters: { status: 'Active' },
  orderBy: 'creation desc',
  start: 0,
  pageLength: 20,
  auto: true,
  cache: ['tenants', 'active'],
  url: 'meuapp.api.get_tenants', // opcional: método customizado
  transform(data) { return data },
  onSuccess(data) {},
  onError(error) {},
  insert: { onSuccess() {}, onError() {} },
  delete: { onSuccess() {}, onError() {} },
  setValue: { onSuccess() {}, onError() {} },
})
```

**API do list resource:**
```js
tenants.data              // array de registros
tenants.originalData      // dados antes do transform
tenants.reload()          // recarrega a lista
tenants.next()            // próxima página
tenants.hasNextPage       // boolean
tenants.update({ filters: { status: 'Inactive' } })

tenants.list.loading      // loading da lista
tenants.list.error
tenants.list.promise

tenants.fetchOne.submit(name)           // busca e atualiza um registro na lista
tenants.setValue.submit({ name, status: 'Closed', description: '...' })
tenants.insert.submit({ description: 'Novo registro' })
tenants.delete.submit(name)
tenants.runDocMethod.submit({ method: 'send_email', name, email: 'x@x.com' })
```

---

### 2.3 createDocumentResource — documento único

```js
import { createDocumentResource } from 'frappe-ui'

let tenant = createDocumentResource({
  doctype: 'Tenant',
  name: tenantName,          // pode ser ref reativa
  whitelistedMethods: {
    activar: 'activate',     // habilita tenant.activar.submit()
    sendEmail: 'send_email',
  },
  transform(doc) { return doc },
  onSuccess(data) {},
  onError(error) {},
  delete: { onSuccess() {}, onError() {} },
  setValue: { onSuccess() {}, onError() {} },
})
```

**API do document resource:**
```js
tenant.doc                 // objeto do documento com todos os campos
tenant.reload()
tenant.update({ name: 'outro-nome' })

tenant.get.loading
tenant.get.error
tenant.get.promise

tenant.setValue.submit({ status: 'Active', email: 'novo@email.com' })
tenant.setValueDebounced.submit({ description: '...' }) // debounced 500ms
tenant.delete.submit()

// Métodos whitelist (se definidos em whitelistedMethods)
tenant.activar.submit()
tenant.activar.loading
tenant.sendEmail.submit({ email: 'x@y.com' })
```

---

## 3. Componentes

### Button
```vue
<Button variant="solid" theme="blue" size="sm" @click="save">Salvar</Button>
<Button variant="outline" :loading="saving" loadingText="Salvando...">Enviar</Button>
<Button variant="ghost" theme="red" icon="trash-2" tooltip="Excluir" />
<Button icon-left="plus" label="Adicionar" />
<Button :disabled="!valid" type="submit" />
```
- **variant:** `subtle`(padrão) | `solid` | `outline` | `ghost`
- **theme:** `gray`(padrão) | `blue` | `green` | `red` | `orange`
- **size:** `sm`(padrão) | `md` | `lg`
- **Slots:** `prefix`, `icon`, `default`, `suffix`

---

### TextInput
```vue
<TextInput v-model="form.email" type="email" label="E-mail" placeholder="nome@empresa.com"
  :required="true" size="sm" variant="subtle" :debounce="300" />
```
- **type:** `text` | `email` | `number` | `password` | `search` | `tel` | `url` | `date` | `time`
- **variant:** `subtle`(padrão) | `outline` | `ghost`
- **size:** `sm`(padrão) | `md` | `lg` | `xl`
- **Slots:** `prefix` (esquerda), `suffix` (direita) — útil para ícones e avatares

---

### Textarea
```vue
<Textarea v-model="form.notes" label="Observações" placeholder="..."
  :rows="4" size="sm" variant="subtle" :debounce="300" />
```
- **variant:** `subtle`(padrão) | `outline`
- **size:** `sm` | `md` | `lg` | `xl`

---

### Select
```vue
<Select v-model="form.tenant_type" placeholder="Selecione..."
  :options="[
    { label: 'Pessoa Física', value: 'Individual' },
    { label: 'Empresa', value: 'Company' },
  ]"
  size="sm" variant="subtle"
/>
```
- **Slots:** `prefix`, `suffix`, `option` (customiza cada opção), `footer`

---

### Combobox (autocomplete/busca)
```vue
<Combobox v-model="selected" :options="options" placeholder="Buscar..."
  :open-on-focus="true" variant="subtle" />
```
- **options:** array de `{ label, value }` ou com grupos `{ group, items: [...] }`
- **Emits:** `update:modelValue`, `update:selectedOption`, `focus`, `blur`, `input`

---

### Checkbox
```vue
<Checkbox v-model="form.accepted" label="Aceito os termos" size="sm" :padding="true" />
```
- **modelValue:** `boolean | 0 | 1`
- **size:** `sm`(padrão) | `md`

---

### Switch
```vue
<Switch v-model="form.active" label="Ativo" description="Habilita o acesso do tenant"
  size="sm" :disabled="false" />
```
- **Emits:** `update:modelValue`, `change`

---

### DatePicker
```vue
<DatePicker v-model="form.birth_date" label="Data de Nascimento"
  placeholder="Selecione a data" variant="subtle" :clearable="true" />
<!-- DateTimePicker e DateRangePicker também disponíveis -->
```

---

### Dialog
```vue
<Dialog v-model="showDialog" :options="{ title: 'Confirmar exclusão', size: 'sm' }"
  :disable-outside-click-to-close="false">
  <template #body-content>
    Tem certeza que deseja excluir este tenant?
  </template>
  <template #actions="{ close }">
    <Button variant="outline" @click="close">Cancelar</Button>
    <Button variant="solid" theme="red" @click="confirmDelete">Excluir</Button>
  </template>
</Dialog>
```
- **Slots:** `body`, `body-header`, `body-title`, `body-content`, `actions` (expõe `{ close }`)
- **Emits:** `update:modelValue`, `close`, `after-leave`

---

### Dropdown
```vue
<Dropdown :options="[
  { label: 'Editar', icon: 'edit', onClick: () => edit() },
  { label: 'Excluir', icon: 'trash-2', theme: 'red', onClick: () => del() },
]" :button="{ label: 'Opções', variant: 'subtle' }" placement="left" />
```
- Suporte a **grupos**, **submenus** e **switches**
- **Slot `default`:** trigger customizado `{ open, close }`
- **Slot `item`:** renderização customizada `{ item, close }`

---

### Alert
```vue
<Alert v-model="showAlert" title="Operação realizada!" theme="green"
  variant="subtle" description="O tenant foi criado com sucesso." :dismissable="true" />
```
- **theme:** `green` | `red` | `blue` | `orange` | `gray`
- **variant:** `subtle`(padrão) | `outline`
- **Slots:** `icon`, `description`, `footer`
- **Emits:** `update:modelValue`, `dismiss`

---

### Badge
```vue
<Badge label="Ativo" theme="green" variant="subtle" size="md" />
<Badge theme="red" variant="solid">Inativo</Badge>
```
- **theme:** `gray`(padrão) | `blue` | `red` | `green` | `orange`
- **variant:** `subtle`(padrão) | `solid` | `outline` | `ghost`
- **Slots:** `prefix`, `default`, `suffix`

---

### Avatar
```vue
<Avatar image="https://..." label="João Silva" size="md" shape="circle" />
```
- **size:** `xs` | `sm` | `md`(padrão) | `lg` | `xl` | `2xl` | `3xl`
- **shape:** `circle`(padrão) | `square`
- **Slots:** `default` (conteúdo customizado), `indicator` (badge no canto inferior direito)

---

### Breadcrumbs
```vue
<Breadcrumbs :items="[
  { label: 'Home', route: '/' },
  { label: 'Tenants', route: '/tenants' },
  { label: 'Novo Tenant' },
]" />
```

---

### ListView
```vue
<ListView
  :columns="[
    { label: 'Nome', key: 'name' },
    { label: 'Tipo', key: 'tenant_type' },
    { label: 'Status', key: 'status' },
  ]"
  :rows="tenants.data"
  row-key="name"
  @row-click="(row) => router.push(`/tenant/${row.name}`)"
>
  <!-- Customizar célula -->
  <template #cell="{ row, column }">
    <Badge v-if="column.key === 'status'" :theme="row.status === 'Active' ? 'green' : 'red'"
      :label="row.status" />
    <span v-else>{{ row[column.key] }}</span>
  </template>
  <!-- Estado vazio -->
  <template #emptyState>
    <div class="text-center text-ink-gray-4">Nenhum tenant encontrado</div>
  </template>
</ListView>
```
- Suporte a **grupos de linhas** via prop `groups`

---

### Tabs
```vue
<Tabs v-model:tab="activeTab" :tabs="[
  { label: 'Dados Gerais', name: 'general', icon: 'user' },
  { label: 'Endereço', name: 'address', icon: 'map-pin' },
]" :vertical="false">
  <template #tab-panel="{ tab }">
    <GeneralTab v-if="tab.name === 'general'" />
    <AddressTab v-if="tab.name === 'address'" />
  </template>
</Tabs>
```

---

### Sidebar
```vue
<Sidebar
  :header="{ title: 'Meu App', logo: '/logo.svg' }"
  :sections="[
    {
      label: 'Principal',
      items: [
        { label: 'Dashboard', icon: 'home', route: '/' },
        { label: 'Tenants', icon: 'users', route: '/tenants' },
      ]
    }
  ]"
  v-model:collapsed="sidebarCollapsed"
/>
```
- **Slots:** `header-logo`, `sidebar-item`, `footer-items`

---

### Tooltip
```vue
<Tooltip text="Excluir este registro" placement="top" :hover-delay="0.5">
  <Button icon="trash-2" variant="ghost" theme="red" />
</Tooltip>
```
- **placement:** `top`(padrão) | `bottom` | `left` | `right`

---

### Progress
```vue
<Progress :value="75" />
```

---

### Slider
```vue
<Slider v-model="form.rating" :min="0" :max="100" />
```

---

### Switch, Rating, Password, Popover, MultiSelect, MonthPicker, TimePicker
Todos seguem o mesmo padrão: `v-model` + props de `size`, `variant`, `label`, `disabled`.
Consulte https://ui.frappe.io/docs/components para props completas de cada um.

---

## 4. Design System

### Background Color (prefixo `bg-surface-`)
```
bg-surface-white
bg-surface-gray-1  até  bg-surface-gray-7
bg-surface-red-1   até  bg-surface-red-7
bg-surface-green-1 até  bg-surface-green-3
bg-surface-blue-1  até  bg-surface-blue-3
bg-surface-amber-1 até  bg-surface-amber-3
bg-surface-orange-1 | bg-surface-violet-1 | bg-surface-cyan-1 | bg-surface-pink-1
bg-surface-menu-bar | bg-surface-cards | bg-surface-modal | bg-surface-selected
```

### Text Color (prefixo `text-ink-`)
```
text-ink-white
text-ink-gray-1 (#EDEDED)  até  text-ink-gray-9 (#171717)
text-ink-red-1  até  text-ink-red-4
text-ink-green-1 até  text-ink-green-3
text-ink-blue-1  até  text-ink-blue-3  |  text-ink-blue-link
text-ink-amber-1 até  text-ink-amber-3
text-ink-cyan-1 | text-ink-pink-1 | text-ink-violet-1
```

### Border Color (prefixo `border-outline-`)
```
border-outline-white
border-outline-gray-1 (#EDEDED)  até  border-outline-gray-5 (#383838)
border-outline-red-1  até  border-outline-red-3
border-outline-green-1 | border-outline-green-2
border-outline-blue-1 | border-outline-amber-1 | border-outline-amber-2 | border-outline-orange-1
```

### Sombras
```
shadow-none | shadow-sm | shadow-md | shadow-lg | shadow-xl | shadow-2xl
```

### Border Radius
```
rounded-none (0px) | rounded-sm (0.25rem) | rounded (0.5rem)
rounded-md (0.625rem) | rounded-lg (0.75rem) | rounded-xl (1rem)
rounded-2xl (1.25rem)
```

---

## 5. Utilitários

### debounce
```js
import { debounce } from 'frappe-ui'
const debouncedSearch = debounce((value) => search(value), 500)
```

### fileToBase64
```js
import { fileToBase64 } from 'frappe-ui'
const base64 = fileToBase64(file) // file deve ser instância de File
```

### pageMetaPlugin — atualiza document.title reativamente
```js
// main.js
import { pageMetaPlugin } from 'frappe-ui'
app.use(pageMetaPlugin)
```
```vue
<!-- Page.vue (Options API) -->
<script>
export default {
  pageMeta() {
    return { title: 'Tenants', emoji: '🏢' }
  }
}
</script>
```

---

## 6. Directives

### v-on-outside-click
```vue
<script>
import { onOutsideClickDirective } from 'frappe-ui'
export default {
  directives: { onOutsideClick: onOutsideClickDirective },
  methods: { close() { this.open = false } }
}
</script>
<template>
  <div v-on-outside-click="close">...</div>
</template>
```

### v-visibility (IntersectionObserver)
```vue
<script>
import { visibilityDirective } from 'frappe-ui'
export default {
  directives: { visibility: visibilityDirective },
  methods: {
    onVisible(visible, entry) { this.visible = visible }
  }
}
</script>
<template>
  <div v-visibility="onVisible">...</div>
</template>
```

---

## 7. Padrões Práticos

### Formulário completo com submit e feedback
```vue
<script setup>
import { reactive } from 'vue'
import { createResource } from 'frappe-ui'

const form = reactive({ tenant_type: '', email: '', full_name: '' })

const createTenant = createResource({
  url: 'meuapp.api.create_tenant',
  onSuccess() {
    // redirecionar ou mostrar feedback
  },
  onError(e) {
    console.error(e)
  }
})
</script>

<template>
  <Select v-model="form.tenant_type" :options="[
    { label: 'Pessoa Física', value: 'Individual' },
    { label: 'Empresa', value: 'Company' },
  ]" />
  <TextInput v-model="form.email" type="email" label="E-mail" />
  <Button variant="solid" :loading="createTenant.loading"
    @click="createTenant.submit(form)">
    Criar Tenant
  </Button>
</template>
```

### ListView + ListResource integrados
```vue
<script setup>
import { createListResource } from 'frappe-ui'
import { useRouter } from 'vue-router'

const router = useRouter()
const tenants = createListResource({
  doctype: 'Tenant',
  fields: ['name', 'tenant_type', 'email', 'status'],
  orderBy: 'creation desc',
  pageLength: 20,
  auto: true,
})
</script>

<template>
  <ListView
    :columns="[
      { label: 'Nome', key: 'name' },
      { label: 'Tipo', key: 'tenant_type' },
      { label: 'E-mail', key: 'email' },
    ]"
    :rows="tenants.data || []"
    row-key="name"
    @row-click="(row) => router.push(`/tenants/${row.name}`)"
  />
  <Button v-if="tenants.hasNextPage" @click="tenants.next()" :loading="tenants.list.loading">
    Carregar mais
  </Button>
</template>
```

### DocumentResource com whitelistedMethods
```vue
<script setup>
import { createDocumentResource } from 'frappe-ui'

const tenant = createDocumentResource({
  doctype: 'Tenant',
  name: route.params.name,
  whitelistedMethods: {
    activate: 'activate',
    sendWelcomeEmail: 'send_welcome_email',
  },
})
</script>

<template>
  <div v-if="tenant.doc">
    <h1>{{ tenant.doc.full_name }}</h1>
    <Button @click="tenant.activate.submit()" :loading="tenant.activate.loading">
      Ativar
    </Button>
    <Button @click="tenant.setValue.submit({ status: 'Inactive' })">
      Desativar
    </Button>
  </div>
</template>
```

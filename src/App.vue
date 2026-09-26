<script setup lang="ts">
import { computed, reactive, ref, watch } from "vue";

type Driver = {
  id: string;
  name: string;
  zones: string[];
};

type Vehicle = {
  id: string;
  plate: string;
  maxLoad: number;
  driverId: string;
};

type Task = {
  id: string;
  plate: string;
  driver: string;
  zone: string;
  task: string;
  weight: number;
  start: string;
  end: string;
  status: string;
  notes: string;
  createdAt: string;
  postponedFrom?: { start: string; end: string };
};

type StoreShape = {
  vehicles: Vehicle[];
  drivers: Driver[];
  tasks: Task[];
};

const project = {
  title: "车辆调度小工具",
  subtitle: "按区域筛车、校验载重，自动检测同一车牌的时间冲突并顺延排班。",
  industry: "物流",
  stack: ["Vue3", "Vite", "TypeScript", "Pinia", "Naive UI"],
  storageKey: "dfwlfront-3-dispatch-v2",
  formTitle: "新增配送任务",
  primaryAction: "校验并排班",
  zones: ["城北", "城东", "城南"],
  statuses: ["待执行", "执行中", "已完成"],
  metricLabels: ["车辆总数", "任务总数", "执行中"]
} as const;

const zones = [...project.zones];
const statuses = [...project.statuses];
const zoneFilters = ["全部区域", ...zones];

function at(hour: number, minute = 0, dayOffset = 0) {
  const d = new Date();
  d.setDate(d.getDate() + dayOffset);
  d.setHours(hour, minute, 0, 0);
  return d.toISOString();
}

function seedStore(): StoreShape {
  const drivers: Driver[] = [
    { id: "drv-1", name: "董飞", zones: ["城北", "城东"] },
    { id: "drv-2", name: "周航", zones: ["城东", "城南"] },
    { id: "drv-3", name: "林岚", zones: ["城北", "城南"] }
  ];
  const vehicles: Vehicle[] = [
    { id: "veh-1", plate: "沪A-82L6", maxLoad: 8, driverId: "drv-1" },
    { id: "veh-2", plate: "沪B-73K9", maxLoad: 5, driverId: "drv-2" },
    { id: "veh-3", plate: "沪C-56M2", maxLoad: 3, driverId: "drv-3" }
  ];
  const tasks: Task[] = [
    {
      id: "task-1",
      plate: "沪A-82L6",
      driver: "董飞",
      zone: "城北",
      task: "商超补货",
      weight: 4,
      start: at(9),
      end: at(11),
      status: "执行中",
      notes: "可立即派车",
      createdAt: at(8, 0, -1)
    },
    {
      id: "task-2",
      plate: "沪B-73K9",
      driver: "周航",
      zone: "城东",
      task: "医药配送",
      weight: 2,
      start: at(13),
      end: at(15, 30),
      status: "待执行",
      notes: "预计17:30返回",
      createdAt: at(8, 30, -1)
    }
  ];
  return { vehicles, drivers, tasks };
}

function loadStore(): StoreShape {
  const raw = localStorage.getItem(project.storageKey);
  if (!raw) return seedStore();
  try {
    const parsed = JSON.parse(raw) as StoreShape;
    if (!parsed.vehicles || !parsed.drivers || !parsed.tasks) return seedStore();
    return parsed;
  } catch {
    return seedStore();
  }
}

const store = loadStore();
const vehicles = ref<Vehicle[]>(store.vehicles);
const drivers = ref<Driver[]>(store.drivers);
const tasks = ref<Task[]>(store.tasks);

function persist() {
  localStorage.setItem(
    project.storageKey,
    JSON.stringify({ vehicles: vehicles.value, drivers: drivers.value, tasks: tasks.value })
  );
}

function driverOf(vehicle: Vehicle) {
  return drivers.value.find((d) => d.id === vehicle.driverId);
}

function fmt(value: string) {
  const d = new Date(value);
  if (Number.isNaN(d.getTime())) return value;
  const pad = (n: number) => String(n).padStart(2, "0");
  return `${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}`;
}

// ---------- 新增任务表单 ----------

const form = reactive({
  zone: "",
  plate: "",
  weight: 1,
  start: "",
  end: "",
  task: "",
  note: ""
});

const formError = ref("");
const formNotice = ref("");

// 第一步：按区域筛车——司机可跑区域包含配送区域的车辆才可选
const eligibleVehicles = computed(() => {
  if (!form.zone) return [];
  return vehicles.value.filter((vehicle) => driverOf(vehicle)?.zones.includes(form.zone));
});

watch(
  () => form.zone,
  () => {
    if (!eligibleVehicles.value.some((v) => v.plate === form.plate)) form.plate = "";
  }
);

// 第二步：同一车牌的时间重叠检测，冲突时顺移到全部冲突任务之后
function resolveConflict(plate: string, start: number, end: number) {
  const duration = end - start;
  const others = tasks.value
    .filter((t) => t.plate === plate && t.status !== "已完成")
    .map((t) => ({ task: t, start: Date.parse(t.start), end: Date.parse(t.end) }))
    .sort((a, b) => a.start - b.start);
  const conflicts = new Map<string, Task>();
  let s = start;
  let e = end;
  let moved = true;
  while (moved) {
    moved = false;
    for (const other of others) {
      if (s < other.end && other.start < e) {
        conflicts.set(other.task.id, other.task);
        s = other.end;
        e = s + duration;
        moved = true;
      }
    }
  }
  return { start: s, end: e, conflicts: [...conflicts.values()] };
}

function fail(message: string) {
  formError.value = message;
  formNotice.value = "";
}

function submit() {
  formError.value = "";
  formNotice.value = "";

  if (!form.zone) return fail("请选择配送区域。");
  if (eligibleVehicles.value.length === 0) {
    return fail(`区域「${form.zone}」没有司机可跑，暂无可用车辆，任务未写入。`);
  }
  const vehicle = vehicles.value.find((v) => v.plate === form.plate);
  if (!vehicle) return fail("请选择车辆。");

  const driver = driverOf(vehicle);
  if (!driver || !driver.zones.includes(form.zone)) {
    return fail(`司机${driver?.name ?? "未登记"}不可跑「${form.zone}」，任务未写入。`);
  }
  if (!(form.weight > 0)) return fail("请填写货物重量。");
  if (form.weight > vehicle.maxLoad) {
    return fail(`货物 ${form.weight} 吨超过 ${vehicle.plate} 的核载上限 ${vehicle.maxLoad} 吨，任务未写入。`);
  }
  if (!form.start || !form.end) return fail("请填写计划起止时间。");
  const s = Date.parse(form.start);
  const e = Date.parse(form.end);
  if (Number.isNaN(s) || Number.isNaN(e)) return fail("计划起止时间格式不正确。");
  if (e <= s) return fail("计划结束时间需晚于开始时间。");
  if (!form.task.trim()) return fail("请填写配送任务内容。");

  const resolved = resolveConflict(vehicle.plate, s, e);
  const postponed = resolved.start !== s;

  tasks.value = [
    ...tasks.value,
    {
      id: crypto.randomUUID(),
      plate: vehicle.plate,
      driver: driver.name,
      zone: form.zone,
      task: form.task.trim(),
      weight: form.weight,
      start: new Date(resolved.start).toISOString(),
      end: new Date(resolved.end).toISOString(),
      status: statuses[0],
      notes: form.note.trim() || "暂无备注",
      createdAt: new Date().toISOString(),
      postponedFrom: postponed
        ? { start: new Date(s).toISOString(), end: new Date(e).toISOString() }
        : undefined
    }
  ];
  persist();

  if (postponed) {
    const names = resolved.conflicts.map((t) => `「${t.task}」`).join("、");
    formNotice.value =
      `${vehicle.plate} 在 ${fmt(new Date(s).toISOString())}–${fmt(new Date(e).toISOString())} 与 ${names} 冲突，` +
      `已顺延至 ${fmt(new Date(resolved.start).toISOString())}–${fmt(new Date(resolved.end).toISOString())} 排入。`;
  } else {
    formNotice.value = `已排入 ${vehicle.plate}：${fmt(new Date(resolved.start).toISOString())}–${fmt(new Date(resolved.end).toISOString())}。`;
  }

  Object.assign(form, { zone: "", plate: "", weight: 1, start: "", end: "", task: "", note: "" });
}

// ---------- 车辆 / 司机档案 ----------

const mgmtError = ref("");
const vehicleForm = reactive({ plate: "", maxLoad: 5, driverId: "" });
const driverForm = reactive({ name: "", zones: [] as string[] });

function addVehicle() {
  mgmtError.value = "";
  const plate = vehicleForm.plate.trim();
  if (!plate) return (mgmtError.value = "请填写车牌号。");
  if (vehicles.value.some((v) => v.plate === plate)) return (mgmtError.value = `车牌 ${plate} 已存在。`);
  if (!(vehicleForm.maxLoad > 0)) return (mgmtError.value = "请填写载重上限（吨）。");
  if (!vehicleForm.driverId) return (mgmtError.value = "请为车辆指定司机。");
  vehicles.value = [
    ...vehicles.value,
    { id: crypto.randomUUID(), plate, maxLoad: vehicleForm.maxLoad, driverId: vehicleForm.driverId }
  ];
  Object.assign(vehicleForm, { plate: "", maxLoad: 5, driverId: "" });
  persist();
}

function removeVehicle(vehicle: Vehicle) {
  mgmtError.value = "";
  if (tasks.value.some((t) => t.plate === vehicle.plate)) {
    return (mgmtError.value = `${vehicle.plate} 名下还有任务，请先删除任务再移除车辆。`);
  }
  vehicles.value = vehicles.value.filter((v) => v.id !== vehicle.id);
  persist();
}

function addDriver() {
  mgmtError.value = "";
  const name = driverForm.name.trim();
  if (!name) return (mgmtError.value = "请填写司机姓名。");
  if (drivers.value.some((d) => d.name === name)) return (mgmtError.value = `司机 ${name} 已存在。`);
  if (driverForm.zones.length === 0) return (mgmtError.value = "请至少勾选一个可跑区域。");
  drivers.value = [...drivers.value, { id: crypto.randomUUID(), name, zones: [...driverForm.zones] }];
  Object.assign(driverForm, { name: "", zones: [] });
  persist();
}

function removeDriver(driver: Driver) {
  mgmtError.value = "";
  if (vehicles.value.some((v) => v.driverId === driver.id)) {
    return (mgmtError.value = `司机 ${driver.name} 已绑定车辆，请先调整车辆。`);
  }
  drivers.value = drivers.value.filter((d) => d.id !== driver.id);
  persist();
}

// ---------- 排班展示 ----------

const zoneFilter = ref(zoneFilters[0]);

const groupedTasks = computed(() =>
  vehicles.value
    .map((vehicle) => ({
      vehicle,
      driver: driverOf(vehicle),
      items: tasks.value
        .filter((t) => t.plate === vehicle.plate)
        .filter((t) => zoneFilter.value === "全部区域" || t.zone === zoneFilter.value)
        .sort((a, b) => Date.parse(a.start) - Date.parse(b.start))
    }))
    .filter((group) => zoneFilter.value === "全部区域" || group.items.length > 0)
);

const metrics = computed(() => [
  vehicles.value.length,
  tasks.value.length,
  tasks.value.filter((t) => t.status === "执行中").length
]);

const chartRows = computed(() =>
  statuses.map((status) => ({
    status,
    value: tasks.value.filter((t) => t.status === status).length
  }))
);

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function flow(task: Task) {
  task.status = statuses[(statuses.indexOf(task.status) + 1) % statuses.length];
  persist();
}

function removeTask(id: string) {
  tasks.value = tasks.value.filter((t) => t.id !== id);
  persist();
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">{{ project.industry }}行业前端最小闭环</p>
          <h1>{{ project.title }}</h1>
          <p class="subtitle">{{ project.subtitle }}</p>
        </div>
        <div class="stack">
          <span v-for="item in project.stack" :key="item" class="tag">{{ item }}</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in project.metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <div class="side">
          <form class="panel" @submit.prevent="submit">
            <h2>{{ project.formTitle }}</h2>
            <div class="form-grid">
              <label>
                配送区域
                <select v-model="form.zone" required>
                  <option value="">请选择</option>
                  <option v-for="zone in zones" :key="zone">{{ zone }}</option>
                </select>
              </label>
              <label>
                车牌号（按区域筛选）
                <select v-model="form.plate" :disabled="!form.zone" required>
                  <option value="">
                    {{ form.zone ? (eligibleVehicles.length ? "请选择" : "该区域暂无可用车辆") : "请先选择区域" }}
                  </option>
                  <option v-for="vehicle in eligibleVehicles" :key="vehicle.id" :value="vehicle.plate">
                    {{ vehicle.plate }}（核载 {{ vehicle.maxLoad }} 吨 / {{ driverOf(vehicle)?.name }}）
                  </option>
                </select>
              </label>
              <label>
                货物重量（吨）
                <input v-model.number="form.weight" type="number" min="0.1" step="0.1" required />
              </label>
              <label>
                计划开始时间
                <input v-model="form.start" type="datetime-local" required />
              </label>
              <label>
                计划结束时间
                <input v-model="form.end" type="datetime-local" required />
              </label>
              <label>
                配送任务
                <input v-model="form.task" type="text" placeholder="如：商超补货" required />
              </label>
              <label>
                备注
                <textarea v-model="form.note" placeholder="填写处理说明或现场备注" />
              </label>
              <p v-if="formError" class="alert alert-error">{{ formError }}</p>
              <p v-if="formNotice" class="alert alert-ok">{{ formNotice }}</p>
              <button type="submit">{{ project.primaryAction }}</button>
            </div>
          </form>

          <section class="panel">
            <h2>车辆与司机档案</h2>
            <div class="form-grid">
              <div class="mgmt-block">
                <h3>车辆（载重上限）</h3>
                <ul class="mgmt-list">
                  <li v-for="vehicle in vehicles" :key="vehicle.id">
                    <span>
                      {{ vehicle.plate }} · 核载 {{ vehicle.maxLoad }} 吨 ·
                      {{ driverOf(vehicle)?.name ?? "未绑定司机" }}
                    </span>
                    <button class="danger small" type="button" @click="removeVehicle(vehicle)">移除</button>
                  </li>
                </ul>
                <div class="mgmt-form">
                  <input v-model="vehicleForm.plate" placeholder="车牌号" />
                  <input v-model.number="vehicleForm.maxLoad" type="number" min="0.5" step="0.5" placeholder="载重上限(吨)" />
                  <select v-model="vehicleForm.driverId">
                    <option value="">选择司机</option>
                    <option v-for="driver in drivers" :key="driver.id" :value="driver.id">{{ driver.name }}</option>
                  </select>
                  <button type="button" @click="addVehicle">添加车辆</button>
                </div>
              </div>

              <div class="mgmt-block">
                <h3>司机（可跑区域）</h3>
                <ul class="mgmt-list">
                  <li v-for="driver in drivers" :key="driver.id">
                    <span>{{ driver.name }} · 可跑 {{ driver.zones.join("、") }}</span>
                    <button class="danger small" type="button" @click="removeDriver(driver)">移除</button>
                  </li>
                </ul>
                <div class="mgmt-form">
                  <input v-model="driverForm.name" placeholder="司机姓名" />
                  <div class="check-row">
                    <label v-for="zone in zones" :key="zone" class="check-item">
                      <input v-model="driverForm.zones" type="checkbox" :value="zone" /> {{ zone }}
                    </label>
                  </div>
                  <button type="button" @click="addDriver">添加司机</button>
                </div>
              </div>
              <p v-if="mgmtError" class="alert alert-error">{{ mgmtError }}</p>
            </div>
          </section>
        </div>

        <section class="list-panel">
          <div class="toolbar">
            <h2>车辆排班（按车牌 · 时间先后）</h2>
            <select v-model="zoneFilter">
              <option v-for="item in zoneFilters" :key="item">{{ item }}</option>
            </select>
          </div>

          <div class="record-grid">
            <div v-if="groupedTasks.length === 0" class="empty">暂无匹配数据</div>
            <section v-for="group in groupedTasks" :key="group.vehicle.id" class="vehicle-group">
              <header class="vehicle-head">
                <strong>{{ group.vehicle.plate }}</strong>
                <span>
                  {{ group.driver?.name ?? "未绑定司机" }} · 核载 {{ group.vehicle.maxLoad }} 吨 ·
                  可跑 {{ group.driver?.zones.join("、") ?? "-" }}
                </span>
              </header>
              <div v-if="group.items.length === 0" class="empty slim">暂无任务</div>
              <article v-for="(task, index) in group.items" :key="task.id" class="record">
                <div class="record-head">
                  <p class="record-title">#{{ index + 1 }} {{ task.task }}</p>
                  <span class="status">{{ task.status }}</span>
                </div>
                <div class="details">
                  <span>计划: {{ fmt(task.start) }} ~ {{ fmt(task.end) }}</span>
                  <span>区域: {{ task.zone }}</span>
                  <span>货物: {{ task.weight }} 吨</span>
                  <span>司机: {{ task.driver }}</span>
                </div>
                <p v-if="task.postponedFrom" class="postponed">
                  原计划 {{ fmt(task.postponedFrom.start) }} ~ {{ fmt(task.postponedFrom.end) }}，因时间冲突顺延至当前时段
                </p>
                <p class="note">{{ task.notes }}</p>
                <div class="actions">
                  <button type="button" @click="flow(task)">流转状态</button>
                  <button class="danger" type="button" @click="removeTask(task.id)">删除</button>
                </div>
              </article>
            </section>
          </div>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>
        </section>
      </section>
    </div>
  </main>
</template>

<script setup lang="ts">
import { computed, reactive, ref } from "vue";

/* ------------------------------ 基础常量与类型 ------------------------------ */

const ZONES = ["城北", "城东", "城南"] as const;
const STATUSES = ["空闲", "执行中", "已完成"] as const;
const STORAGE_KEY = "dfwlfront-3-dispatch-v2";
const LEGACY_KEY = "dfwlfront-3-dispatch";

interface Driver {
  id: string;
  name: string;
  zones: string[]; // 可跑区域
}

interface Vehicle {
  id: string;
  plate: string; // 车牌号
  capacity: number; // 载重上限（吨）
  driverId: string;
}

interface PostponeInfo {
  fromStart: string; // 原计划开始 ISO
  fromEnd: string; // 原计划结束 ISO
  toStart: string; // 顺延后开始 ISO
  toEnd: string; // 顺延后结束 ISO
}

interface DispatchTask {
  id: string;
  plate: string;
  driver: string;
  zone: string;
  title: string;
  weight: number; // 货物重量（吨）
  start: string; // 计划开始 ISO
  end: string; // 计划结束 ISO
  status: string;
  notes: string;
  postponed: PostponeInfo | null;
  createdAt: string;
}

interface Store {
  drivers: Driver[];
  vehicles: Vehicle[];
  tasks: DispatchTask[];
}

interface Feedback {
  type: "error" | "warn" | "ok";
  text: string;
}

function uid(): string {
  return typeof crypto !== "undefined" && "randomUUID" in crypto
    ? crypto.randomUUID()
    : `id-${Date.now()}-${Math.random().toString(36).slice(2)}`;
}

/* --------------------------------- 时间工具 --------------------------------- */

const pad = (n: number) => String(n).padStart(2, "0");

/** Date -> datetime-local 输入框值（本地时区） */
function toLocalInput(d: Date): string {
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}T${pad(d.getHours())}:${pad(d.getMinutes())}`;
}

/** ISO -> “MM-DD HH:mm” 展示 */
function fmtDateTime(iso: string): string {
  const d = new Date(iso);
  return `${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}`;
}

/** 顺延时长描述 */
function delayText(info: PostponeInfo): string {
  const minutes = Math.round((new Date(info.toStart).getTime() - new Date(info.fromStart).getTime()) / 60000);
  if (minutes <= 0) return "0 分钟";
  const h = Math.floor(minutes / 60);
  const m = minutes % 60;
  if (h === 0) return `${m} 分钟`;
  return m === 0 ? `${h} 小时` : `${h} 小时 ${m} 分钟`;
}

function defaultSlot(startOffsetMin: number, durationMin: number): { start: string; end: string } {
  const d = new Date(Date.now() + startOffsetMin * 60000);
  d.setMinutes(d.getMinutes() + (30 - (d.getMinutes() % 30)) % 30, 0, 0);
  return { start: toLocalInput(d), end: toLocalInput(new Date(d.getTime() + durationMin * 60000)) };
}

/** 用今天的时钟生成种子时间，便于直接演示冲突顺延 */
function todayAt(hours: number, minutes = 0): string {
  const d = new Date();
  d.setHours(hours, minutes, 0, 0);
  return d.toISOString();
}

/* --------------------------------- 种子数据 --------------------------------- */

function seedStore(): Store {
  const drivers: Driver[] = [
    { id: "seed-driver-1", name: "董飞", zones: ["城北", "城东"] },
    { id: "seed-driver-2", name: "周航", zones: ["城东"] },
    { id: "seed-driver-3", name: "徐丽", zones: ["城南", "城北"] }
  ];
  const vehicles: Vehicle[] = [
    { id: "seed-vehicle-1", plate: "沪A-82L6", capacity: 5, driverId: "seed-driver-1" },
    { id: "seed-vehicle-2", plate: "沪B-73K9", capacity: 3, driverId: "seed-driver-2" },
    { id: "seed-vehicle-3", plate: "沪C-15T2", capacity: 8, driverId: "seed-driver-3" }
  ];
  const tasks: DispatchTask[] = [
    {
      id: "seed-task-1",
      plate: "沪A-82L6",
      driver: "董飞",
      zone: "城北",
      title: "商超补货",
      weight: 4,
      start: todayAt(8, 30),
      end: todayAt(10, 0),
      status: "执行中",
      notes: "可立即派车",
      postponed: null,
      createdAt: todayAt(8, 0)
    },
    {
      id: "seed-task-2",
      plate: "沪B-73K9",
      driver: "周航",
      zone: "城东",
      title: "医药配送",
      weight: 2.2,
      start: todayAt(9, 0),
      end: todayAt(10, 30),
      status: "执行中",
      notes: "预计 10:30 返回站点",
      postponed: null,
      createdAt: todayAt(8, 10)
    },
    {
      id: "seed-task-3",
      plate: "沪A-82L6",
      driver: "董飞",
      zone: "城北",
      title: "冷链蔬菜",
      weight: 3.5,
      start: todayAt(13, 0),
      end: todayAt(15, 0),
      status: "空闲",
      notes: "午间冷链专线",
      postponed: null,
      createdAt: todayAt(8, 20)
    }
  ];
  return { drivers, vehicles, tasks };
}

/* ----------------------------- 旧版数据（v1）迁移 ----------------------------- */

function migrateLegacy(): Store | null {
  const raw = localStorage.getItem(LEGACY_KEY);
  if (!raw) return null;
  try {
    const oldRecords = JSON.parse(raw) as Array<Record<string, unknown>>;
    if (!Array.isArray(oldRecords)) return null;
    const drivers: Driver[] = [];
    const vehicles: Vehicle[] = [];
    const tasks: DispatchTask[] = [];
    oldRecords.forEach((record, index) => {
      const plate = String(record.vehicle ?? "").trim();
      const driverName = String(record.driver ?? "").trim();
      const zone = String(record.zone ?? "").trim();
      if (!plate) return;
      let driver = drivers.find((item) => item.name === driverName);
      if (!driver && driverName) {
        driver = { id: `mig-driver-${index + 1}`, name: driverName, zones: zone ? [zone] : [] };
        drivers.push(driver);
      }
      if (!vehicles.some((item) => item.plate === plate)) {
        vehicles.push({
          id: `mig-vehicle-${index + 1}`,
          plate,
          capacity: 5, // 旧数据没有载重信息，迁移后给默认值，可在车辆记录里修改
          driverId: driver?.id ?? ""
        });
      }
      const createdAt = typeof record.createdAt === "string" ? record.createdAt : new Date().toISOString();
      tasks.push({
        id: `mig-task-${index + 1}`,
        plate,
        driver: driverName,
        zone,
        title: String(record.task ?? "未命名任务"),
        weight: 0,
        start: createdAt,
        end: new Date(new Date(createdAt).getTime() + 90 * 60000).toISOString(),
        status: STATUSES.includes(record.status as (typeof STATUSES)[number]) ? String(record.status) : "空闲",
        notes: typeof record.notes === "string" ? record.notes : "由旧版数据迁移",
        postponed: null,
        createdAt
      });
    });
    return { drivers, vehicles, tasks };
  } catch {
    return null;
  }
}

function loadStore(): Store {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (raw) {
    try {
      const parsed = JSON.parse(raw) as Store;
      if (parsed && Array.isArray(parsed.tasks)) {
        return { drivers: parsed.drivers ?? [], vehicles: parsed.vehicles ?? [], tasks: parsed.tasks };
      }
    } catch {
      // 数据损坏时落到种子数据
    }
  }
  return migrateLegacy() ?? seedStore();
}

const initialStore = loadStore();
const drivers = ref<Driver[]>(initialStore.drivers);
const vehicles = ref<Vehicle[]>(initialStore.vehicles);
const tasks = ref<DispatchTask[]>(initialStore.tasks);

function persist() {
  const store: Store = { drivers: drivers.value, vehicles: vehicles.value, tasks: tasks.value };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(store));
}

/* --------------------------------- 派生数据 --------------------------------- */

const driverMap = computed(() => new Map(drivers.value.map((driver) => [driver.id, driver])));

function driverOf(vehicle: Vehicle | undefined): Driver | undefined {
  return vehicle ? driverMap.value.get(vehicle.driverId) : undefined;
}

/* -------------------------------- 新增任务表单 -------------------------------- */

const slot = defaultSlot(60, 90);
const form = reactive({
  zone: "",
  plate: "",
  title: "",
  weight: "",
  start: slot.start,
  end: slot.end,
  notes: ""
});
const feedback = ref<Feedback | null>(null);

/** 第一步：按区域筛车 —— 只保留绑定司机可跑该区域的车辆 */
const candidateVehicles = computed(() => {
  if (!form.zone) return [];
  return vehicles.value.filter((vehicle) => driverOf(vehicle)?.zones.includes(form.zone));
});

const matchedVehicle = computed(() =>
  vehicles.value.find((vehicle) => vehicle.plate === form.plate.trim())
);
const matchedDriver = computed(() => driverOf(matchedVehicle.value));

function choosePlate(plate: string) {
  form.plate = plate;
}

/**
 * 第二步：同车牌时间重叠检测。
 * 有冲突就整体顺延到该车全部冲突任务之后（区间 [start, end) 相接不算冲突），
 * 若顺延后又与后续任务重叠，则继续顺延，直到不再冲突。
 */
function resolveSchedule(plate: string, start: Date, end: Date) {
  let nextStart = start.getTime();
  let nextEnd = end.getTime();
  const conflicts: DispatchTask[] = [];
  for (;;) {
    const hit = tasks.value.find(
      (task) =>
        task.plate === plate &&
        nextStart < new Date(task.end).getTime() &&
        new Date(task.start).getTime() < nextEnd
    );
    if (!hit) break;
    conflicts.push(hit);
    const hitEnd = new Date(hit.end).getTime();
    const duration = nextEnd - nextStart;
    nextStart = hitEnd;
    nextEnd = hitEnd + duration;
  }
  return { start: new Date(nextStart), end: new Date(nextEnd), conflicts };
}

function reject(text: string) {
  feedback.value = { type: "error", text };
}

function submit() {
  feedback.value = null;
  const zone = form.zone;
  const plate = form.plate.trim();
  const title = form.title.trim();
  const weight = Number(form.weight);

  if (!zone || !plate || !title || !form.start || !form.end) {
    reject("请完整填写配送区域、车牌号、配送任务和计划起止时间。");
    return;
  }
  const reqStart = new Date(form.start);
  const reqEnd = new Date(form.end);
  if (Number.isNaN(reqStart.getTime()) || Number.isNaN(reqEnd.getTime()) || reqEnd <= reqStart) {
    reject("计划结束时间必须晚于开始时间，请重新选择起止时间。内容已保留。");
    return;
  }
  if (!Number.isFinite(weight) || weight < 0) {
    reject("货物重量需为不小于 0 的数字（吨）。内容已保留。");
    return;
  }

  const vehicle = vehicles.value.find((item) => item.plate === plate);
  if (!vehicle) {
    reject(`车牌 ${plate} 不在车辆记录中，请先在下方“车辆记录”里登记车辆和载重上限。内容已保留，未写入排班。`);
    return;
  }

  // 区域校验：车辆绑定司机的可跑区域必须覆盖所选区域
  const driver = driverOf(vehicle);
  if (!driver) {
    reject(`车辆 ${plate} 未绑定有效司机，无法核对可跑区域。内容已保留，未写入排班。`);
    return;
  }
  if (!driver.zones.includes(zone)) {
    reject(
      `区域不符：车辆 ${plate} 绑定司机「${driver.name}」的可跑区域为 ${driver.zones.join("、") || "（未设置）"}，` +
        `不覆盖${zone}。内容已保留，请更换区域或车辆，未写入排班。`
    );
    return;
  }

  // 载重校验：货物重量不能超过车辆载重上限
  if (weight > vehicle.capacity) {
    reject(
      `载重不符：货物 ${weight} 吨超过车辆 ${plate} 的载重上限 ${vehicle.capacity} 吨。` +
        `内容已保留，请减重或更换车辆，未写入排班。`
    );
    return;
  }

  // 同车牌时间重叠检测 + 自动顺延
  const resolved = resolveSchedule(plate, reqStart, reqEnd);
  const delayed = resolved.start.getTime() !== reqStart.getTime();

  const newTask: DispatchTask = {
    id: uid(),
    plate,
    driver: driver.name,
    zone,
    title,
    weight,
    start: resolved.start.toISOString(),
    end: resolved.end.toISOString(),
    status: STATUSES[0],
    notes: form.notes.trim() || "暂无备注",
    postponed: delayed
      ? {
          fromStart: reqStart.toISOString(),
          fromEnd: reqEnd.toISOString(),
          toStart: resolved.start.toISOString(),
          toEnd: resolved.end.toISOString()
        }
      : null,
    createdAt: new Date().toISOString()
  };
  tasks.value = [newTask, ...tasks.value];
  persist();

  if (delayed) {
    const conflictText = resolved.conflicts
      .map((task) => `「${task.title} ${fmtDateTime(task.start)}~${fmtDateTime(task.end)}」`)
      .join("、");
    const info = newTask.postponed as PostponeInfo;
    feedback.value = {
      type: "warn",
      text:
        `车牌 ${plate} 的计划区间 ${fmtDateTime(info.fromStart)}~${fmtDateTime(info.fromEnd)} 与同车任务 ${conflictText} 时间重叠，` +
        `已排在该车全部冲突任务之后，顺延至 ${fmtDateTime(info.toStart)}~${fmtDateTime(info.toEnd)}（顺延 ${delayText(info)}）。任务已保存。`
    };
  } else {
    feedback.value = { type: "ok", text: `排班成功：${plate} 将于 ${fmtDateTime(newTask.start)}~${fmtDateTime(newTask.end)} 执行「${title}」。` };
  }

  form.title = "";
  form.weight = "";
  form.notes = "";
}

/* -------------------------------- 列表与统计 -------------------------------- */

const zoneView = ref("全部区域");
const plateView = ref("全部车牌");

const plateOptions = computed(() =>
  [...new Set(tasks.value.map((task) => task.plate))].sort((a, b) => a.localeCompare(b, "zh"))
);

/** 重新打开后按车牌分组、同车按计划开始时间先后展示 */
const groupedTasks = computed(() => {
  const map = new Map<string, DispatchTask[]>();
  tasks.value
    .filter((task) => zoneView.value === "全部区域" || task.zone === zoneView.value)
    .filter((task) => plateView.value === "全部车牌" || task.plate === plateView.value)
    .slice()
    .sort((a, b) => a.start.localeCompare(b.start))
    .forEach((task) => {
      const list = map.get(task.plate);
      if (list) list.push(task);
      else map.set(task.plate, [task]);
    });
  return [...map.entries()]
    .sort((a, b) => a[0].localeCompare(b[0], "zh"))
    .map(([plate, items]) => ({ plate, items }));
});

const metrics = computed(() => [
  { label: "车辆总数", value: vehicles.value.length },
  { label: "司机人数", value: drivers.value.length },
  { label: "排班任务", value: tasks.value.length },
  { label: "顺延任务", value: tasks.value.filter((task) => task.postponed).length }
]);

const chartRows = computed(() =>
  STATUSES.map((status) => ({
    status,
    value: tasks.value.filter((task) => task.status === status).length
  }))
);
const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function nextStatus(status: string) {
  const index = STATUSES.indexOf(status as (typeof STATUSES)[number]);
  return STATUSES[(index + 1) % STATUSES.length];
}

function flow(task: DispatchTask) {
  task.status = nextStatus(task.status);
  persist();
}

function removeTask(id: string) {
  tasks.value = tasks.value.filter((task) => task.id !== id);
  persist();
}

/* -------------------------------- 司机登记 -------------------------------- */

const driverForm = reactive({ name: "", zones: [] as string[] });
const driverMsg = ref<Feedback | null>(null);

function toggleDriverZone(zone: string) {
  const index = driverForm.zones.indexOf(zone);
  if (index >= 0) driverForm.zones.splice(index, 1);
  else driverForm.zones.push(zone);
}

function addDriver() {
  driverMsg.value = null;
  const name = driverForm.name.trim();
  if (!name) {
    driverMsg.value = { type: "error", text: "请填写司机姓名。" };
    return;
  }
  if (drivers.value.some((driver) => driver.name === name)) {
    driverMsg.value = { type: "error", text: `司机「${name}」已存在，姓名不可重复。` };
    return;
  }
  if (driverForm.zones.length === 0) {
    driverMsg.value = { type: "error", text: "请至少勾选一个可跑区域。" };
    return;
  }
  drivers.value = [...drivers.value, { id: uid(), name, zones: [...driverForm.zones] }];
  persist();
  driverMsg.value = { type: "ok", text: `已新增司机「${name}」。` };
  driverForm.name = "";
  driverForm.zones = [];
}

function removeDriver(id: string) {
  drivers.value = drivers.value.filter((driver) => driver.id !== id);
  persist();
}

/* -------------------------------- 车辆登记 -------------------------------- */

const vehicleForm = reactive({ plate: "", capacity: "", driverId: "" });
const vehicleMsg = ref<Feedback | null>(null);

function addVehicle() {
  vehicleMsg.value = null;
  const plate = vehicleForm.plate.trim();
  const capacity = Number(vehicleForm.capacity);
  if (!plate) {
    vehicleMsg.value = { type: "error", text: "请填写车牌号。" };
    return;
  }
  if (vehicles.value.some((vehicle) => vehicle.plate === plate)) {
    vehicleMsg.value = { type: "error", text: `车牌 ${plate} 已登记，不可重复。` };
    return;
  }
  if (!Number.isFinite(capacity) || capacity <= 0) {
    vehicleMsg.value = { type: "error", text: "载重上限需为大于 0 的数字（吨）。" };
    return;
  }
  if (!vehicleForm.driverId) {
    vehicleMsg.value = { type: "error", text: "请选择绑定司机（司机的可跑区域决定可配送范围）。" };
    return;
  }
  vehicles.value = [...vehicles.value, { id: uid(), plate, capacity, driverId: vehicleForm.driverId }];
  persist();
  const driverName = driverMap.value.get(vehicleForm.driverId)?.name ?? "";
  vehicleMsg.value = { type: "ok", text: `已登记车辆 ${plate}（载重上限 ${capacity} 吨，司机 ${driverName}）。` };
  vehicleForm.plate = "";
  vehicleForm.capacity = "";
  vehicleForm.driverId = "";
}

function removeVehicle(id: string) {
  vehicles.value = vehicles.value.filter((vehicle) => vehicle.id !== id);
  persist();
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">物流行业前端最小闭环</p>
          <h1>车辆调度小工具</h1>
          <p class="subtitle">
            维护车辆载重上限与司机可跑区域；新建任务先按区域筛车，再检查同车牌时间重叠，冲突自动顺延并记录顺延区间，排班保存在浏览器本地。
          </p>
        </div>
        <div class="stack">
          <span class="tag">Vue3</span>
          <span class="tag">Vite</span>
          <span class="tag">TypeScript</span>
          <span class="tag">localStorage</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="item in metrics" :key="item.label" class="metric">
          <span>{{ item.label }}</span>
          <strong>{{ item.value }}</strong>
        </article>
      </section>

      <section class="workspace">
        <!-- 新增任务：区域筛车 + 载重/区域校验 + 冲突顺延 -->
        <form class="panel task-form" @submit.prevent="submit">
          <h2>新增配送任务</h2>
          <div class="form-grid">
            <label>
              配送区域
              <select v-model="form.zone" required>
                <option value="">请选择区域</option>
                <option v-for="zone in ZONES" :key="zone" :value="zone">{{ zone }}</option>
              </select>
            </label>

            <label>
              车牌号（先按区域筛选）
              <input
                v-model="form.plate"
                type="text"
                list="plate-candidates"
                placeholder="选择或输入车牌"
                required
              />
              <datalist id="plate-candidates">
                <option v-for="vehicle in candidateVehicles" :key="vehicle.id" :value="vehicle.plate" />
              </datalist>
              <div v-if="!form.zone" class="hint">请先选择配送区域，再挑选可跑该区域的车辆。</div>
              <ul v-else-if="candidateVehicles.length" class="candidates">
                <li
                  v-for="vehicle in candidateVehicles"
                  :key="vehicle.id"
                  :class="{ active: form.plate === vehicle.plate }"
                  @click="choosePlate(vehicle.plate)"
                >
                  {{ vehicle.plate }} · 载重上限 {{ vehicle.capacity }} 吨 ·
                  司机 {{ driverOf(vehicle)?.name }}（{{ driverOf(vehicle)?.zones.join("/") }}）
                </li>
              </ul>
              <div v-else class="hint warn-text">当前区域暂无可跑车辆，请先在下方登记司机区域或车辆绑定。</div>
            </label>

            <label>
              司机
              <input :value="matchedDriver?.name ?? ''" type="text" readonly placeholder="选择车牌后自动带出" />
            </label>

            <label>
              配送任务
              <input v-model="form.title" type="text" placeholder="如：商超补货" required />
            </label>

            <label>
              货物重量（吨）
              <input v-model="form.weight" type="number" min="0" step="0.1" placeholder="如：3.5" required />
              <div v-if="matchedVehicle" class="hint">
                {{ matchedVehicle.plate }} 载重上限 {{ matchedVehicle.capacity }} 吨，
                可跑区域：{{ matchedDriver?.zones.join("、") || "未设置" }}
              </div>
            </label>

            <div class="time-range">
              <label>
                计划开始
                <input v-model="form.start" type="datetime-local" required />
              </label>
              <label>
                计划结束
                <input v-model="form.end" type="datetime-local" required />
              </label>
            </div>

            <label>
              备注
              <textarea v-model="form.notes" placeholder="填写处理说明或现场备注" />
            </label>

            <div v-if="feedback" class="feedback" :class="feedback.type">{{ feedback.text }}</div>

            <button type="submit">分配任务</button>
          </div>
        </form>

        <!-- 排班列表：按车牌分组、同车按时间先后 -->
        <section class="list-panel">
          <div class="toolbar">
            <h2>排班列表（按车牌）</h2>
            <div class="toolbar-filters">
              <select v-model="plateView">
                <option value="全部车牌">全部车牌</option>
                <option v-for="plate in plateOptions" :key="plate" :value="plate">{{ plate }}</option>
              </select>
              <select v-model="zoneView">
                <option value="全部区域">全部区域</option>
                <option v-for="zone in ZONES" :key="zone" :value="zone">{{ zone }}</option>
              </select>
            </div>
          </div>

          <div v-if="groupedTasks.length === 0" class="empty">暂无匹配排班</div>
          <div v-else class="plate-groups">
            <article v-for="group in groupedTasks" :key="group.plate" class="plate-group">
              <div class="plate-head">
                <span class="plate-name">{{ group.plate }}</span>
                <span class="plate-meta">共 {{ group.items.length }} 个任务 · 司机 {{ group.items[0].driver }}</span>
              </div>
              <div class="task-seq">
                <div v-for="(task, index) in group.items" :key="task.id" class="record">
                  <div class="seq-line">
                    <span class="seq-no">{{ index + 1 }}</span>
                    <span class="seq-time">{{ fmtDateTime(task.start)}} ~ {{ fmtDateTime(task.end) }}</span>
                  </div>
                  <div class="record-head">
                    <p class="record-title">{{ task.title }}</p>
                    <span class="status">{{ task.status }}</span>
                  </div>
                  <div class="details">
                    <span>配送区域: {{ task.zone }}</span>
                    <span>司机: {{ task.driver }}</span>
                    <span>货物重量: {{ task.weight }} 吨</span>
                    <span v-if="task.postponed" class="postponed-chip">已顺延</span>
                  </div>
                  <div v-if="task.postponed" class="postponed">
                    顺延区间：{{ fmtDateTime(task.postponed.fromStart) }} ~ {{ fmtDateTime(task.postponed.fromEnd) }}
                    → {{ fmtDateTime(task.postponed.toStart) }} ~ {{ fmtDateTime(task.postponed.toEnd) }}
                    （顺延 {{ delayText(task.postponed) }}，排在同车冲突任务之后）
                  </div>
                  <p class="note">{{ task.notes }}</p>
                  <div class="actions">
                    <button type="button" @click="flow(task)">流转状态</button>
                    <button class="danger" type="button" @click="removeTask(task.id)">删除</button>
                  </div>
                </div>
              </div>
            </article>
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

      <!-- 车辆 / 司机登记 -->
      <section class="registry-grid">
        <section class="panel">
          <h2>车辆记录（载重上限）</h2>
          <table class="registry-table">
            <thead>
              <tr>
                <th>车牌号</th>
                <th>载重上限</th>
                <th>绑定司机</th>
                <th>可跑区域</th>
                <th></th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="vehicles.length === 0">
                <td colspan="5" class="empty-cell">暂无车辆，请先登记</td>
              </tr>
              <tr v-for="vehicle in vehicles" :key="vehicle.id">
                <td>{{ vehicle.plate }}</td>
                <td>{{ vehicle.capacity }} 吨</td>
                <td>{{ driverOf(vehicle)?.name ?? "未绑定" }}</td>
                <td>{{ driverOf(vehicle)?.zones.join("、") ?? "—" }}</td>
                <td>
                  <button class="danger tiny" type="button" @click="removeVehicle(vehicle.id)">删除</button>
                </td>
              </tr>
            </tbody>
          </table>
          <div class="mini-form">
            <div class="mini-row">
              <label>
                车牌号
                <input v-model="vehicleForm.plate" type="text" placeholder="如：沪D-22M8" />
              </label>
              <label>
                载重上限（吨）
                <input v-model="vehicleForm.capacity" type="number" min="0" step="0.1" placeholder="如：5" />
              </label>
            </div>
            <label>
              绑定司机
              <select v-model="vehicleForm.driverId">
                <option value="">请选择司机</option>
                <option v-for="driver in drivers" :key="driver.id" :value="driver.id">
                  {{ driver.name }}（{{ driver.zones.join("、") || "未设置区域" }}）
                </option>
              </select>
            </label>
            <div v-if="vehicleMsg" class="feedback" :class="vehicleMsg.type">{{ vehicleMsg.text }}</div>
            <button type="button" @click="addVehicle">登记车辆</button>
          </div>
        </section>

        <section class="panel">
          <h2>司机记录（可跑区域）</h2>
          <div v-if="drivers.length === 0" class="empty">暂无司机，请先登记</div>
          <ul v-else class="driver-list">
            <li v-for="driver in drivers" :key="driver.id">
              <div class="driver-info">
                <strong>{{ driver.name }}</strong>
                <span class="zone-chips">
                  <i v-for="zone in driver.zones" :key="zone" class="chip">{{ zone }}</i>
                  <em v-if="driver.zones.length === 0">未设置可跑区域</em>
                </span>
              </div>
              <button class="danger tiny" type="button" @click="removeDriver(driver.id)">删除</button>
            </li>
          </ul>
          <div class="mini-form">
            <label>
              司机姓名
              <input v-model="driverForm.name" type="text" placeholder="如：林晨" />
            </label>
            <div class="check-group">
              <span class="check-label">可跑区域：</span>
              <label v-for="zone in ZONES" :key="zone" class="check-item">
                <input type="checkbox" :checked="driverForm.zones.includes(zone)" @change="toggleDriverZone(zone)" />
                {{ zone }}
              </label>
            </div>
            <div v-if="driverMsg" class="feedback" :class="driverMsg.type">{{ driverMsg.text }}</div>
            <button type="button" @click="addDriver">登记司机</button>
          </div>
        </section>
      </section>
    </div>
  </main>
</template>

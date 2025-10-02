<!-- components/ThreeBars.vue -->
<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, watch, computed } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { RoomEnvironment } from "three/examples/jsm/environments/RoomEnvironment.js";
import { RoundedBoxGeometry } from "three/examples/jsm/geometries/RoundedBoxGeometry.js";
import { scaleLinear, scaleSequential, interpolateTurbo } from "d3";

type Datum = { label: string; value: number };

const props = defineProps<{
  data: Datum[];
  width?: number;
  height?: number;
  barWidth?: number;
  gap?: number;
  maxBarHeight?: number;
  background?: string;
  autoRotate?: boolean;
}>();

const host = ref<HTMLDivElement | null>(null);
let renderer: THREE.WebGLRenderer | null = null;// الرسّام WebGL
let scene: THREE.Scene | null = null;// المشهد
let camera: THREE.PerspectiveCamera | null = null;// الكاميرا
let controls: OrbitControls | null = null;// جهزة التحكم
let animationId: number | null = null;// معرف طلب الإطار للرسوم المتحركة
let ro: ResizeObserver | null = null;// مراقب تغيير الحجم
let raycaster: THREE.Raycaster | null = null;// كاشفver
let pointer: THREE.Vector2 | null = null;// متجه ثنائي الأبعاد لتخزين إحداثيات المؤشر
let tooltipDiv: HTMLDivElement | null = null;// عنصر div لتلميح الأدوات

const bg = computed(() => props.background ?? "#0b1220"); // أزرق غامق
const h = computed(() => props.height ?? 520);
const bw = computed(() => props.barWidth ?? 0.9);
const gap = computed(() => props.gap ?? 0.5);
const maxBarH = computed(() => props.maxBarHeight ?? 10);

function buildScene() {
  if (!host.value) return;

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true }); // إنشاء الرسّام WebGL مع تفعيل تنعيم الحواف (antialias).
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); //ضبط دقة الرسم حسب شاشة المستخدم لكن بحد أقصى ×2 لتقليل استهلاك الموارد.
  renderer.setSize(host.value.clientWidth, host.value.clientHeight);// ضبط حجم الرسم ليملأ العنصر المضيف.
  renderer.setClearColor(new THREE.Color(bg.value));// تعيين لون الخلفية.
  renderer.shadowMap.enabled = true;// تفعيل خريطة الظلال.
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;// تعيين نوع خريطة الظلال إلى PCFSoftShadowMap للحصول على ظلال ناعمة.
  renderer.toneMapping = THREE.ACESFilmicToneMapping;// تعيين نوع معالجة النغمة إلى ACESFilmicToneMapping لتحسين نطاق الألوان والسطوع.
  renderer.toneMappingExposure = 1.05;// ضبط تعريض معالجة النغمة.
  renderer.physicallyCorrectLights = true;// تفعيل الأضواء الفيزيائية الصحيحة لتحسين واقعية الإضاءة.
  host.value.appendChild(renderer.domElement);// إضافة عنصر الرسم إلى العنصر المضيف في DOM.

  // Scene + Env
  scene = new THREE.Scene();// إنشاء مشهد جديد.
  scene.fog = new THREE.Fog(bg.value, 25, 60);// إضافة تأثير ضبابي للمشهد.

  const pmrem = new THREE.PMREMGenerator(renderer);// إنشاء مولد خرائط البيئة مسبقة التصفية.
  const env = pmrem.fromScene(new RoomEnvironment() // إنشاء مشهد غرفة.
  , 0.04// مستوى التفاصيل (0.04 يعني تفاصيل عالية
).texture;// إنشاء خريطة بيئية من مشهد غرفة.
  scene.environment = env;// تعيين خريطة بيئة للمشهد.


  // Camera
  camera = new THREE.PerspectiveCamera(
    60, // زاوية الرؤية العمودية
    host.value.clientWidth / host.value.clientHeight, // نسبة العرض إلى الارتفاع
    0.1, // أقرب مسافة للرؤية
    200 // أبعد مسافة للرؤية 
  );// إنشاء كاميرا منظور.
  camera.position.set(10, 10, 16);// تعيين موقع الكاميرا في الفضاء ثلاثي الأبعاد.

  // Lights (ناعمة)
  const hemi = new THREE.HemisphereLight(0xffffff, 0x334455, 0.6);//إنشاء ناعمة سماء.
  scene.add(hemi);// إضافة ضوء نصف كروي إلى المشهد.
  const key = new THREE.DirectionalLight(0xffffff, 2.2);// إنشاء ضوء اتجاهي (ضوء رئيسي).
  key.position.set(8, 12, 6);// تعيين موقع الضوء الرئيسي.
  key.castShadow = true;// تفعيل إلقاء الظلال للضوء الرئيسي.
  key.shadow.mapSize.set(1024, 1024);// ضبط حجم خريطة الظلال.
  key.shadow.camera.near = 1;// ضبط أقرب مسافة لظل الكاميرا.
  key.shadow.camera.far = 60;// ضبط أبعد مسافة لظل الكاميرا.
  scene.add(key);// إضافة الضوء الرئيسي إلى المشهد.

  // Controls
  controls = new OrbitControls(camera!, renderer.domElement);//إنشاء جهزة التحكم.
  controls.enableDamping = true;// تفعيل التخميد لجعل الحركة أكثر سلاسة.
  controls.autoRotate = props.autoRotate ?? true;// تفعيل الدوران التلقائي إذا لم يتم تحديده في الخصائص.
  controls.autoRotateSpeed = 0.8;// تعيين سرعة الدوران التلقائي.
  controls.target.set(0, 2.2, 0);// تعيين نقطة الهدف التي تدور حولها الكاميرا.

  // Floor + Grid
  const floorMat = new THREE.MeshStandardMaterial({
    color: 0x0f172a,
    roughness: 1,
    metalness: 0,
  });// إنشاء مادة قياسية للأرضية.
  const floor = new THREE.Mesh(new THREE.PlaneGeometry(120, 120), floorMat);// إنشاء شبكة أرضية كبيرة.
  floor.receiveShadow = true;// تفعيل استقبال الظلال للأرضية.
  floor.rotation.x = -Math.PI / 2;// تدوير الأرضية لتكون أفقية.
  scene.add(floor);// إضافة العرض إلى المشهد.


  const grid = new THREE.GridHelper(120, 60, 0x1f3b5c, 0x12253f); // خطوط قابلة بالكاد
  (grid.material as THREE.Material).opacity = 0.45;// ضبط شفافية الخطوط.
  (grid.material as THREE.Material).transparent = true;// تفعيل الشفافية للخطوط.
  scene.add(grid);// إضافة الخطوط إلى المشهد.



  // Tooltip (HTML)
  tooltipDiv = document.createElement("div");// إنشاء عنصر div جديد ليكون تلميحًا.
  tooltipDiv.className =
    "pointer-events-none absolute px-2 py-1 text-sm rounded-xl shadow bg-black/80 text-white opacity-0 transition-opacity";
  Object.assign(tooltipDiv.style, {
    position: "absolute",
    transform: "translate(-50%, -120%)",
  });// تعيين بعض الأنماط الأساسية للتلميح.
  host.value.style.position = "relative";
  host.value.appendChild(tooltipDiv);// إضافة التلميح إلى العنصر المضيف.

  // Raycaster
  raycaster = new THREE.Raycaster();// إنشاء كاشف الأشعة للتفاعل مع الأجسام في المشهد.
  pointer = new THREE.Vector2();// إنشاء متجه ثنائي الأبعاد لتخزين إحداثيات المؤشر.
  renderer.domElement.addEventListener("pointermove", onPointerMove);// إضافة مستمع حدث لتحريك المؤشر.

  // Bars أول رسم
  createOrUpdateBars(true);// إنشاء أو تحديث الأعمدة لأول مرة.

  // Resize
  ro = new ResizeObserver(onResize);// إنشاء مراقب تغيير الحجم.
  ro.observe(host.value);// مراقبة تغييرات حجم العنصر المضيف لإعادة ضبط العرض والكاميرا.

  animate();
}// بناء المشهد ثلاثي الأبعاد.

function onResize() {
  if (!host.value || !renderer || !camera) return;
  const width = host.value.clientWidth;
  const height = host.value.clientHeight;
  renderer.setSize(width, height);//
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
}// إعادة ضبط حجم العرض ونسبة العرض إلى الارتفاع للكاميرا عند تغيير حجم العنصر المضيف.

function onPointerMove(event: PointerEvent) {
  if (!renderer || !host.value || !pointer) return;
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
  checkHover(event);
}// تحديث إحداثيات المؤشر بناءً على موقعه في العنصر المضيف والتحقق من التفاعل مع الأعمدة.

function barContainer() {
  // نجمع الأعمدة تحت Group واحد لترتيب أسهل
  let group = scene!.getObjectByName("bars") as THREE.Group;
  if (!group) {
    group = new THREE.Group();
    group.name = "bars";
    scene!.add(group);
  }
  return group;
}// الحصول على مجموعة الأعمدة أو إنشاؤها إذا لم تكن موجودة.

function createOrUpdateBars(firstTime = false) {
  if (!scene) return;
  const group = barContainer();

  const maxV = Math.max(...props.data.map((d) => d.value), 1);
  const n = props.data.length;
  const yScale = scaleLinear().domain([0, maxV]).range([0.2, maxBarH.value]);
  const colorScale = scaleSequential(interpolateTurbo).domain([0, maxV]);

  const totalWidth = n * bw.value + (n - 1) * gap.value;
  const startX = -totalWidth / 2 + bw.value / 2;

  // خريطة للوصول السريع حسب label
  const existing = new Map<string, THREE.Object3D>();
  group.children.forEach((c) => existing.set(c.userData?.label, c));

  // تحديث/إضافة
  props.data.forEach((d, i) => {
    const targetH = yScale(d.value);
    const x = startX + i * (bw.value + gap.value);
    const z = 0;

    let node = existing.get(d.label) as THREE.Group;
    if (!node) {
      node = new THREE.Group();
      node.position.set(x, 0, z);

      // Rounded bar
      const geom = new RoundedBoxGeometry(bw.value, 1, bw.value, 6, 0.18);
      const mat = new THREE.MeshStandardMaterial({
        color: new THREE.Color(colorScale(d.value)),
        metalness: 0.35,
        roughness: 0.35,
        envMapIntensity: 1.0,
      });

      const mesh = new THREE.Mesh(geom, mat);
      mesh.castShadow = true;
      mesh.receiveShadow = false;
      mesh.position.y = 0.5;
      mesh.userData = { kind: "barMesh" };

      // Edge subtle
      const edges = new THREE.EdgesGeometry(geom);
      const line = new THREE.LineSegments(
        edges,
        new THREE.LineBasicMaterial({ color: 0x0b1220 })
      );
      line.position.copy(mesh.position);
      line.userData = { kind: "barEdge" };

      node.add(mesh);
      node.add(line);
      node.userData = {
        kind: "bar",
        label: d.label,
        value: d.value,
        currentH: 0.2,
        targetH,
      };
      group.add(node);

      // بداية أنيميشن لطيفة
      if (firstTime) node.userData.currentH = 0.2;
    } else {
      node.position.x = x;
      node.userData.value = d.value;
      node.userData.targetH = targetH;
      // غيّر اللون حسب القيمة
      const mesh = node.children.find(
        (c) => c.userData?.kind === "barMesh"
      ) as THREE.Mesh;
      (mesh.material as THREE.MeshStandardMaterial).color.set(
        colorScale(d.value)
      );
    }
  });

  // حذف العناصر التي لم تعد موجودة
  group.children.slice().forEach((c) => {
    const lbl = c.userData?.label;
    if (lbl && !props.data.some((d) => d.label === lbl)) {
      group.remove(c);
      c.traverse((o) => {
        if ((o as any).geometry) (o as any).geometry.dispose?.();
        if ((o as any).material) {
          const m = (o as any).material;
          if (Array.isArray(m)) m.forEach((mm) => mm.dispose?.());
          else m.dispose?.();
        }
      });
    }
  });
}// إنشاء أو تحديث الأعمدة بناءً على البيانات الجديدة.

function applyHeightsLerp(dt: number) {
  const group = barContainer();
  group.children.forEach((node) => {
    if (node.userData?.kind !== "bar") return;
    const mesh = node.children.find(
      (c) => c.userData?.kind === "barMesh"
    ) as THREE.Mesh;
    const edge = node.children.find(
      (c) => c.userData?.kind === "barEdge"
    ) as THREE.LineSegments;
    const cur = node.userData.currentH as number;
    const target = node.userData.targetH as number;
    // Lerp سلس
    const next = THREE.MathUtils.damp(cur, target, 6.0, dt);
    node.userData.currentH = next;

    // حدّث السكيل والموضع (المكعب طوله 1 ومرتكز عند y=0.5)
    mesh.scale.y = next;
    mesh.position.y = (next * 1) / 2;
    edge.scale.y = next;
    edge.position.y = mesh.position.y;
  });
}// تطبيق ارتفاعاً متحركاً للعناصر في المجموعة.


function checkHover(evt: PointerEvent) {
  if (
    !raycaster ||
    !pointer ||
    !camera ||
    !renderer ||
    !scene ||
    !host.value ||
    !tooltipDiv
  )
    return;
  raycaster.setFromCamera(pointer, camera);

  // اجعل التفاعل على الـ mesh فقط
  const bars: THREE.Mesh[] = [];
  barContainer().children.forEach((g) => {
    const m = g.children.find(
      (c) => c.userData?.kind === "barMesh"
    ) as THREE.Mesh;
    if (m) bars.push(m);
  });
  const hits = raycaster.intersectObjects(bars, false);

  // reset glow
  bars.forEach(
    (m) => ((m.material as THREE.MeshStandardMaterial).emissiveIntensity = 0)
  );

  const hit = hits[0];
  if (hit) {
    const mesh = hit.object as THREE.Mesh;
    const parent = mesh.parent as THREE.Group;
    (mesh.material as THREE.MeshStandardMaterial).emissive = new THREE.Color(
      mesh.material["color"]
    );
    (mesh.material as THREE.MeshStandardMaterial).emissiveIntensity = 0.25;

    // Tooltip
    const { label, value } = parent.userData;
    tooltipDiv.textContent = `${label}: ${value}`;
    tooltipDiv.style.opacity = "1";
    const rect = renderer.domElement.getBoundingClientRect();
    tooltipDiv.style.left = `${evt.clientX - rect.left}px`;
    tooltipDiv.style.top = `${evt.clientY - rect.top}px`;
  } else {
    tooltipDiv.style.opacity = "0";
  }
}// التحقق من تفاعل المؤشر مع الأعمدة وتحديث التلميح واللمعان.

let lastT = 0;
function animate(t = 0) {
  if (!renderer || !scene || !camera || !controls) return;
  animationId = requestAnimationFrame(animate);
  const dt = Math.min(0.05, (t - lastT) / 1000); // cap delta
  lastT = t;

  controls.update();
  applyHeightsLerp(dt);
  renderer.render(scene, camera);
}// حلقة الرسوم المتحركة التي تقوم بتحديث المشهد باستمرار.

onMounted(() => buildScene());

watch(
  () => props.data,
  () => createOrUpdateBars(),
  { deep: true }
);// مراقبة تغييرات البيانات لإعادة إنشاء أو تحديث الأعمدة.

onBeforeUnmount(() => {
  if (animationId) cancelAnimationFrame(animationId);
  if (ro) ro.disconnect();
  if (renderer)
    renderer.domElement.removeEventListener("pointermove", onPointerMove);
  tooltipDiv?.remove();
  if (scene) {
    scene.traverse((obj) => {
      if ((obj as any).geometry) (obj as any).geometry.dispose?.();
      if ((obj as any).material) {
        const m = (obj as any).material;
        if (Array.isArray(m)) m.forEach((mm) => mm.dispose?.());
        else m.dispose?.();
      }
    });
  }
  controls?.dispose();
  renderer?.dispose();
  renderer = null;
  scene = null;
  camera = null;
  controls = null;
  raycaster = null;
  pointer = null;
});// تنظيف الموارد عند إلغاء تركيب المكون.
</script>

<template>
  <div
    ref="host"
    class="w-full h-[540px] rounded-2xl overflow-hidden ring-1 ring-white/5"
  ></div>
</template>

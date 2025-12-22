<template>
  <div ref="containerRef" id="comment-dom" :class="['comment-content', 'giscus', { fill }]" />
</template>

<script setup>
const { theme } = useData();

const props = defineProps({
  fill: {
    type: [Boolean, String],
    default: false,
  },
});

const containerRef = ref(null);

const loadGiscus = () => {
  if (!containerRef.value) return;
  containerRef.value.innerHTML = "";
  const el = document.createElement("script");
  el.src = "https://giscus.app/client.js";
  el.async = true;
  el.crossOrigin = "anonymous";

  const option = theme.value.comment?.giscus || {};  
  el.setAttribute("data-repo", option.repo || "");
  el.setAttribute("data-repo-id", option.repoId || "");
  el.setAttribute("data-category", option.category || "");
  el.setAttribute("data-category-id", option.categoryId || "");
  el.setAttribute("data-mapping", option.mapping || "pathname");
  el.setAttribute("data-strict", String(option.strict ?? 0));
  el.setAttribute("data-reactions-enabled", String(option.reactionsEnabled ?? 1));
  el.setAttribute("data-emit-metadata", String(option.emitMetadata ?? 0));
  el.setAttribute("data-input-position", option.inputPosition || "bottom");
  el.setAttribute("data-theme", option.theme || "preferred_color_scheme");
  el.setAttribute("data-lang", option.lang || "zh-CN");
  el.setAttribute("data-loading", "lazy");

  containerRef.value.appendChild(el);
};

onMounted(() => {
  loadGiscus();
});
</script>

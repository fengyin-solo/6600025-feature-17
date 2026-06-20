<script setup lang="ts">
import { computed, watch, ref } from 'vue';
import VChart from 'vue-echarts';
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { LineChart } from 'echarts/charts';
import {
  GridComponent,
  TooltipComponent,
  LegendComponent
} from 'echarts/components';
import { useCanBusStore } from '../store/canbus';

use([CanvasRenderer, LineChart, GridComponent, TooltipComponent, LegendComponent]);

const store = useCanBusStore();
const chartRef = ref<InstanceType<typeof VChart> | null>(null);

const colors = ['#06b6d4', '#22c55e', '#ef4444', '#eab308', '#a855f7'];

const chartOption = computed(() => {
  const signalEntries = Array.from(store.signals.entries());

  const series = signalEntries.map(([name, sig], idx) => ({
    name,
    type: 'line' as const,
    smooth: true,
    symbol: 'none',
    lineStyle: { width: 2 },
    itemStyle: { color: colors[idx % colors.length] },
    data: sig.data.map(d => [d.time, d.value])
  }));

  return {
    backgroundColor: '#111827',
    tooltip: {
      trigger: 'axis',
      backgroundColor: '#1f2937',
      borderColor: '#374151',
      textStyle: { color: '#e5e7eb', fontSize: 12 },
      formatter: (params: any) => {
        if (!Array.isArray(params) || params.length === 0) return '';
        const time = new Date(params[0].value[0]).toLocaleTimeString('zh-CN', { hour12: false });
        let html = `<div style="font-size:11px;color:#9ca3af">${time}</div>`;
        for (const p of params) {
          html += `<div style="display:flex;align-items:center;gap:6px">
            <span style="display:inline-block;width:8px;height:8px;border-radius:50%;background:${p.color}"></span>
            <span>${p.seriesName}: <b>${Number(p.value[1]).toFixed(1)}</b></span>
          </div>`;
        }
        return html;
      }
    },
    legend: {
      top: 8,
      textStyle: { color: '#9ca3af', fontSize: 11 },
      itemWidth: 12,
      itemHeight: 2
    },
    grid: {
      left: 60,
      right: 20,
      top: 45,
      bottom: 35
    },
    xAxis: {
      type: 'value',
      axisLabel: {
        color: '#6b7280',
        fontSize: 10,
        formatter: (val: number) => {
          const d = new Date(val);
          return d.toLocaleTimeString('zh-CN', { hour12: false });
        }
      },
      axisLine: { lineStyle: { color: '#374151' } },
      splitLine: { lineStyle: { color: '#1f2937' } }
    },
    yAxis: {
      type: 'value',
      axisLabel: { color: '#6b7280', fontSize: 10 },
      axisLine: { lineStyle: { color: '#374151' } },
      splitLine: { lineStyle: { color: '#1f2937' } }
    },
    series
  };
});

const signalStatsList = computed(() => {
  const signalEntries = Array.from(store.signals.entries());
  return signalEntries.map(([name], idx) => {
    const stats = store.signalStats.get(name);
    return {
      name,
      color: colors[idx % colors.length],
      min: stats?.min ?? 0,
      max: stats?.max ?? 0,
      avg: stats?.avg ?? 0,
      count: stats?.count ?? 0
    };
  });
});
</script>

<template>
  <div class="flex flex-col h-full bg-gray-900 rounded-lg overflow-hidden">
    <div class="px-4 py-2 bg-gray-800 border-b border-gray-700 flex items-center justify-between">
      <h3 class="text-sm font-semibold text-gray-300">信号趋势图</h3>
      <span class="text-xs text-gray-500">
        {{ store.signals.size }} 个信号活跃
      </span>
    </div>

    <div v-if="signalStatsList.length > 0" class="px-4 py-3 bg-gray-800 border-b border-gray-700">
      <div class="text-xs text-gray-400 mb-2">信号摘要</div>
      <div class="grid grid-cols-1 gap-2">
        <div
          v-for="sig in signalStatsList"
          :key="sig.name"
          class="flex items-center justify-between text-xs"
        >
          <div class="flex items-center gap-2 min-w-0">
            <span
              class="w-2 h-2 rounded-full shrink-0"
              :style="{ backgroundColor: sig.color }"
            ></span>
            <span class="text-gray-300 truncate">{{ sig.name }}</span>
          </div>
          <div class="flex items-center gap-3 shrink-0">
            <div class="text-right">
              <span class="text-gray-500">最小</span>
              <span class="text-cyan-400 font-mono ml-1">{{ sig.min.toFixed(1) }}</span>
            </div>
            <div class="text-right">
              <span class="text-gray-500">最大</span>
              <span class="text-green-400 font-mono ml-1">{{ sig.max.toFixed(1) }}</span>
            </div>
            <div class="text-right">
              <span class="text-gray-500">平均</span>
              <span class="text-yellow-400 font-mono ml-1">{{ sig.avg.toFixed(1) }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="flex-1 p-2 relative">
      <VChart
        ref="chartRef"
        :option="chartOption"
        autoresize
        class="w-full h-full"
        style="min-height: 200px;"
      />
    </div>
    <div v-if="store.signals.size === 0" class="absolute inset-0 flex items-center justify-center pointer-events-none">
      <p class="text-gray-600 text-sm">等待信号数据...</p>
    </div>
  </div>
</template>

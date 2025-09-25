<template>
  <div class="w-full p-4">
    <h2 class="text-xl font-bold mb-4">Таблица - fixed</h2>

    <!-- Main container with horizontal scroll -->
    <div class="relative border rounded-lg overflow-hidden">
      <div class="overflow-x-auto">
        <Table class="min-w-[1200px] w-full">
          <TableHeader>
            <!-- Main Headers Row with Groups -->
            <TableRow>
              <template v-for="headerGroup in table.getHeaderGroups().slice(0, 1)" :key="headerGroup.id">
                <TableHead
                  v-for="header in headerGroup.headers"
                  :key="header.id"
                  :colspan="header.colSpan"
                  :rowspan="header.column.id === 'number' || header.column.id === 'region' || header.column.id === 'jamoaJaharsoni' ? 2 : 1"
                  :class="[
                    'text-center border-r',
                    header.column.columnDef.meta?.fixed === 'left' && header.column.id === 'number' ? 'sticky left-0 z-20 bg-background w-[50px]' : '',
                    header.column.columnDef.meta?.fixed === 'left' && header.column.id === 'region' ? 'sticky left-[50px] z-20 bg-background w-[120px]' : '',
                    header.column.columnDef.meta?.fixed === 'right' && header.column.id === 'shundanGroup' ? 'sticky right-0 z-20 bg-background border-l' : ''
                  ]"
                >
                  <FlexRender
                    :render="header.column.columnDef.header"
                    :props="header.getContext()"
                  />
                </TableHead>
              </template>
            </TableRow>

            <!-- Sub Headers Row -->
            <TableRow>
              <template v-for="headerGroup in table.getHeaderGroups().slice(1, 2)" :key="headerGroup.id">
                <!-- Skip rendering cells for columns that have rowspan=2 -->
                <TableHead
                  v-for="header in headerGroup.headers.filter(h => h.column.id !== 'number' && h.column.id !== 'region' && h.column.id !== 'jamoaJaharsoni')"
                  :key="header.id"
                  :class="[
                    'text-center border-r',
                    header.column.columnDef.meta?.fixed === 'right' && header.column.id === 'shaxsiy' ? 'sticky right-[100px] z-20 bg-background w-[100px]' : '',
                    header.column.columnDef.meta?.fixed === 'right' && header.column.id === 'ijarada' ? 'sticky right-0 z-20 bg-background border-l w-[100px]' : ''
                  ]"
                >
                  <FlexRender
                    v-if="!header.isPlaceholder"
                    :render="header.column.columnDef.header"
                    :props="header.getContext()"
                  />
                </TableHead>
              </template>
            </TableRow>
          </TableHeader>

          <TableBody>
            <TableRow v-for="row in table.getRowModel().rows" :key="row.id">
              <TableCell
                v-for="cell in row.getVisibleCells()"
                :key="cell.id"
                :class="[
                  'text-center border-r',
                  cell.column.columnDef.meta?.fixed === 'left' && cell.column.id === 'number' ? 'sticky left-0 z-10 bg-background w-[50px]' : '',
                  cell.column.columnDef.meta?.fixed === 'left' && cell.column.id === 'region' ? 'sticky left-[50px] z-10 bg-background w-[120px]' : '',
                  cell.column.columnDef.meta?.fixed === 'right' && cell.column.id === 'shaxsiy' ? 'sticky right-[100px] z-10 bg-background w-[100px]' : '',
                  cell.column.columnDef.meta?.fixed === 'right' && cell.column.id === 'ijarada' ? 'sticky right-0 z-10 bg-background border-l w-[100px]' : ''
                ]"
              >
                <FlexRender
                  :render="cell.column.columnDef.cell"
                  :props="cell.getContext()"
                />
              </TableCell>
            </TableRow>

            <!-- Summary Row -->
            <TableRow class="font-semibold bg-muted/50">
              <TableCell class="sticky left-0 z-10 bg-muted/50 border-r w-[50px] text-center">
              </TableCell>
              <TableCell class="sticky left-[50px] z-10 bg-muted/50 border-r w-[120px]">
                Жами:
              </TableCell>
              <TableCell class="border-r text-center">124</TableCell>
              <TableCell class="border-r text-center">982 213</TableCell>
              <TableCell class="border-r text-center">765 927</TableCell>
              <TableCell class="border-r text-center">12 458 084</TableCell>
              <TableCell class="border-r text-center">5 827 211</TableCell>
              <TableCell class="border-r text-center">97.1 %</TableCell>
              <TableCell class="sticky right-[100px] z-10 bg-muted/50 border-r w-[100px] text-center">
                368 919
              </TableCell>
              <TableCell class="sticky right-0 z-10 bg-muted/50 border-l w-[100px] text-center">
                21 089
              </TableCell>
            </TableRow>
          </TableBody>
        </Table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import {
  FlexRender,
  getCoreRowModel,
  useVueTable,
} from '@tanstack/vue-table'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'

// Sample data
const data = ref([
  {
    number: 1,
    region: 'Андижон',
    jamoaJaharsoni: 17,
    maxallalarJami: 405,
    maxallalarShundan: 304,
    xonadonlarStatistik: 298213,
    xonadonlarXeTomonidan: 234107,
    kesmaKecma: 101.2,
    shaxsiy: 9845,
    ijarada: 290
  },
  {
    number: 2,
    region: 'Наманган',
    jamoaJaharsoni: 16,
    maxallalarJami: 911,
    maxallalarShundan: 876,
    xonadonlarStatistik: 345203,
    xonadonlarXeTomonidan: 459028,
    kesmaKecma: 103.1,
    shaxsiy: 3145,
    ijarada: 546
  },
  {
    number: 4,
    region: 'Фаргона',
    jamoaJaharsoni: 13,
    maxallalarJami: 874,
    maxallalarShundan: 567,
    xonadonlarStatistik: 320083,
    xonadonlarXeTomonidan: 425793,
    kesmaKecma: 98.0,
    shaxsiy: 3459,
    ijarada: 978
  },
  {
    number: 3,
    region: 'Жиззах',
    jamoaJaharsoni: 13,
    maxallalarJami: 865,
    maxallalarShundan: 813,
    xonadonlarStatistik: 428084,
    xonadonlarXeTomonidan: 546789,
    kesmaKecma: 96.8,
    shaxsiy: 7129,
    ijarada: 678
  },
  {
    number: 4,
    region: 'Сирдарё',
    jamoaJaharsoni: 12,
    maxallalarJami: 487,
    maxallalarShundan: 423,
    xonadonlarStatistik: 542214,
    xonadonlarXeTomonidan: 189763,
    kesmaKecma: 105.5,
    shaxsiy: 9786,
    ijarada: 124
  },
  {
    number: 5,
    region: 'Самарқанд',
    jamoaJaharsoni: 11,
    maxallalarJami: 678,
    maxallalarShundan: 611,
    xonadonlarStatistik: 876428,
    xonadonlarXeTomonidan: 243508,
    kesmaKecma: 100.1,
    shaxsiy: 1567,
    ijarada: 187
  },
  {
    number: 6,
    region: 'Навоий',
    jamoaJaharsoni: 16,
    maxallalarJami: 488,
    maxallalarShundan: 398,
    xonadonlarStatistik: 420829,
    xonadonlarXeTomonidan: 892743,
    kesmaKecma: 93.1,
    shaxsiy: 2789,
    ijarada: 823
  },
  {
    number: 7,
    region: 'Тошкент',
    jamoaJaharsoni: 10,
    maxallalarJami: 914,
    maxallalarShundan: 875,
    xonadonlarStatistik: 980231,
    xonadonlarXeTomonidan: 342578,
    kesmaKecma: 95.9,
    shaxsiy: 8564,
    ijarada: 981
  }
])

// Column definitions with grouped headers
const columns = [
  {
    id: 'number',
    accessorKey: 'number',
    header: '№',
    cell: ({ row }) => row.getValue('number'),
    meta: {
      fixed: 'left'
    }
  },
  {
    id: 'region',
    accessorKey: 'region',
    header: 'Худуд',
    cell: ({ row }) => row.getValue('region'),
    meta: {
      fixed: 'left'
    }
  },
  {
    id: 'jamoaJaharsoni',
    accessorKey: 'jamoaJaharsoni',
    header: 'Туман, Шахар сони',
    cell: ({ row }) => row.getValue('jamoaJaharsoni'),
  },
  {
    id: 'maxallalarGroup',
    header: 'Махаллалар',
    columns: [
      {
        id: 'maxallalarJami',
        accessorKey: 'maxallalarJami',
        header: 'Жами сони',
        cell: ({ row }) => row.getValue('maxallalarJami').toLocaleString(),
      },
      {
        id: 'maxallalarShundan',
        accessorKey: 'maxallalarShundan',
        header: 'Шундан харитада чизилган',
        cell: ({ row }) => row.getValue('maxallalarShundan').toLocaleString(),
      }
    ]
  },
  {
    id: 'xonadonlarGroup',
    header: 'Хонадонлар сон',
    columns: [
      {
        id: 'xonadonlarStatistik',
        accessorKey: 'xonadonlarStatistik',
        header: 'Статистик маълумотлар',
        cell: ({ row }) => row.getValue('xonadonlarStatistik').toLocaleString(),
      },
      {
        id: 'xonadonlarXeTomonidan',
        accessorKey: 'xonadonlarXeTomonidan',
        header: 'ХЕ томонидан урганилган',
        cell: ({ row }) => row.getValue('xonadonlarXeTomonidan').toLocaleString(),
      },
      {
        id: 'kesmaKecma',
        accessorKey: 'kesmaKecma',
        header: '% кесимида',
        cell: ({ row }) => `${row.getValue('kesmaKecma')} %`,
      }
    ]
  },
  {
    id: 'shundanGroup',
    header: 'Шундан',
    meta: {
      fixed: 'right'
    },
    columns: [
      {
        id: 'shaxsiy',
        accessorKey: 'shaxsiy',
        header: 'Шахсий',
        cell: ({ row }) => row.getValue('shaxsiy').toLocaleString(),
        meta: {
          fixed: 'right'
        }
      },
      {
        id: 'ijarada',
        accessorKey: 'ijarada',
        header: 'Ижарада',
        cell: ({ row }) => row.getValue('ijarada').toLocaleString(),
        meta: {
          fixed: 'right'
        }
      }
    ]
  }
]

// Create table instance
const table = useVueTable({
  data: data.value,
  columns,
  getCoreRowModel: getCoreRowModel(),
})
</script>

<style scoped>
/* Add shadows to fixed columns for better visibility */
:deep(.sticky.left-0::after),
:deep(.sticky.left-\[50px\]::after) {
  content: '';
  position: absolute;
  top: 0;
  right: -3px;
  bottom: 0;
  width: 3px;
  background: linear-gradient(to right, rgba(0,0,0,0.08), transparent);
  pointer-events: none;
}

:deep(.sticky.right-0::before),
:deep(.sticky.right-\[100px\]::before) {
  content: '';
  position: absolute;
  top: 0;
  left: -3px;
  bottom: 0;
  width: 3px;
  background: linear-gradient(to left, rgba(0,0,0,0.08), transparent);
  pointer-events: none;
}

/* Ensure headers have proper background */
:deep(thead th) {
  background: hsl(var(--background));
}

/* Fix z-index layering for sticky headers */
:deep(thead th.sticky) {
  z-index: 30;
}
</style>

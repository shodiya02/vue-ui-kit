<template>
  <div class="relative border rounded-lg overflow-auto max-w-full">
    <Table class="w-full">
      <TableHeader>
        <TableRow v-for="headerGroup in table.getHeaderGroups()" :key="headerGroup.id">
          <TableHead
            v-for="header in headerGroup.headers"
            :key="header.id"
            :class="[
              'text-center border-r bg-background',
              {
                'sticky z-20': header.column.getIsPinned(),
                'border-r-2 border-r-gray-300': header.column.getIsPinned() === 'left',
                'border-l-2': header.column.getIsPinned() === 'right',
              },
            ]"
            :colspan="header.colSpan"
            :style="getHeaderStyle(header)"
          >
            <div v-if="!header.isPlaceholder">
              {{ header.column.columnDef.header }}
            </div>
          </TableHead>
        </TableRow>
      </TableHeader>

      <TableBody>
        <TableRow v-for="row in table.getRowModel().rows" :key="row.id">
          <TableCell
            v-for="cell in row.getVisibleCells()"
            :key="cell.id"
            :class="[
              'text-center border-r bg-background',
              {
                'sticky z-10': cell.column.getIsPinned(),
                'border-r-2 border-r-gray-300': cell.column.getIsPinned() === 'left',
                'border-l-2': cell.column.getIsPinned() === 'right',
              },
            ]"
            :style="getCellStyle(cell)"
          >
            {{ cell.getValue() }}
          </TableCell>
        </TableRow>
      </TableBody>
    </Table>
  </div>
</template>

<script setup>
import { computed, ref, watch } from 'vue'
import { getCoreRowModel, useVueTable } from '@tanstack/vue-table'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'

const props = defineProps({
  data: {
    type: Array,
    required: true,
  },
  columns: {
    type: Array,
    required: true,
  },
  leftFixed: {
    type: Number,
    default: 0,
  },
  rightFixed: {
    type: Number,
    default: 0,
  },
})

// Create column pinning state
const columnPinning = ref({
  left: [],
  right: [],
})

// Transform columns for TanStack Table
const transformedColumns = computed(() => {
  const transformColumn = (col, leafIndex) => {
    // For grouped columns (have children)
    if (col.children && col.children.length > 0) {
      let currentLeafIndex = leafIndex
      const transformedChildren = []

      col.children.forEach((child) => {
        const result = transformColumn(child, currentLeafIndex)
        transformedChildren.push(result.column)
        currentLeafIndex = result.nextIndex
      })

      return {
        column: {
          id: col.id?.toString() || col.dataField || `group_${leafIndex}`,
          header: col.title,
          columns: transformedChildren,
          meta: {
            originalColumn: col,
          },
        },
        nextIndex: currentLeafIndex,
      }
    }

    // For leaf columns (actual data columns)
    return {
      column: {
        id: col.id?.toString() || col.dataField || `col_${leafIndex}`,
        accessorKey: col.dataField,
        header: col.title,
        size: parseInt(col.width) || 150,
        cell: (info) => info.getValue(),
        meta: {
          originalColumn: col,
          leafIndex: leafIndex, // Store the leaf index for pinning
        },
      },
      nextIndex: leafIndex + 1,
    }
  }

  let currentLeafIndex = 0
  const result = []

  props.columns.forEach((col) => {
    const transformed = transformColumn(col, currentLeafIndex)
    result.push(transformed.column)
    currentLeafIndex = transformed.nextIndex
  })

  return result
})

// Get all leaf columns in order
const leafColumns = computed(() => {
  const leaves = []

  const extractLeaves = (columns) => {
    columns.forEach((col) => {
      if (col.columns && col.columns.length > 0) {
        extractLeaves(col.columns)
      } else {
        leaves.push(col)
      }
    })
  }

  extractLeaves(transformedColumns.value)
  return leaves
})

// Calculate column pinning when props change
watch(
  [() => props.leftFixed, () => props.rightFixed, leafColumns],
  () => {
    const leaves = leafColumns.value

    // Set left pinned columns
    const leftPinned = leaves.slice(0, props.leftFixed).map((col) => col.id)

    // Set right pinned columns
    const rightPinned =
      props.rightFixed > 0 ? leaves.slice(-props.rightFixed).map((col) => col.id) : []

    columnPinning.value = {
      left: leftPinned,
      right: rightPinned,
    }
  },
  { immediate: true },
)

// Create the table instance
const table = useVueTable({
  get data() {
    return props.data
  },
  get columns() {
    return transformedColumns.value
  },
  getCoreRowModel: getCoreRowModel(),
  enableColumnPinning: true,
  state: {
    columnPinning: columnPinning.value,
  },
  onColumnPinningChange: (updater) => {
    columnPinning.value = typeof updater === 'function' ? updater(columnPinning.value) : updater
  },
  enableColumnResizing: true,
  columnResizeMode: 'onChange',
})

// Watch for pinning changes and update table state
watch(
  columnPinning,
  (newPinning) => {
    table.setColumnPinning(newPinning)
  },
  { deep: true },
)

// Calculate styles for pinned headers
const getHeaderStyle = (header) => {
  const isPinned = header.column.getIsPinned()
  if (!isPinned) return {}

  const style = {}

  if (isPinned === 'left') {
    style.left = `${header.column.getStart('left')}px`
  } else if (isPinned === 'right') {
    style.right = `${header.column.getAfter('right')}px`
  }

  const width = header.column.columnDef.size || header.column.getSize()
  style.width = `${width}px`
  style.minWidth = `${width}px`

  return style
}

// Calculate styles for pinned cells
const getCellStyle = (cell) => {
  const isPinned = cell.column.getIsPinned()
  if (!isPinned) return {}

  const style = {}

  if (isPinned === 'left') {
    style.left = `${cell.column.getStart('left')}px`
  } else if (isPinned === 'right') {
    style.right = `${cell.column.getAfter('right')}px`
  }

  const width = cell.column.columnDef.size || cell.column.getSize()
  style.width = `${width}px`
  style.minWidth = `${width}px`

  return style
}
</script>

<style scoped>
/* Ensure sticky positioning works properly */
.sticky {
  position: sticky !important;
}

/* Add shadow to pinned columns for better visual separation */
th.sticky.z-20,
td.sticky.z-10 {
  background-color: white;
}

/* Shadow for left pinned columns */
th.sticky.z-20:last-child,
td.sticky.z-10:last-child {
  box-shadow: 2px 0 5px -2px rgba(0, 0, 0, 0.15);
}

/* Shadow for right pinned columns */
th.sticky.z-20:first-child,
td.sticky.z-10:first-child {
  box-shadow: -2px 0 5px -2px rgba(0, 0, 0, 0.15);
}

/* Ensure table has proper layout */
table {
  table-layout: fixed;
}
</style>

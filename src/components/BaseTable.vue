<template>
  <div class="relative border rounded-lg overflow-hidden">
  <Table class="min-w-[1200px] w-full">
    <TableHeader>
      <!-- Dynamic Headers based on depth -->
      <TableRow v-for="(headerGroup, groupIndex) in table.getHeaderGroups()" :key="headerGroup.id">
        <TableHead
          v-for="header in headerGroup.headers"
          :key="header.id"
          :class="getHeaderClasses(header, groupIndex)"
          :colspan="header.colSpan"
          :rowspan="getHeaderRowSpan(header, groupIndex)"
        >
          <FlexRender
            :props="header.getContext()"
            :render="header.column.columnDef.header"
          />
        </TableHead>
      </TableRow>
    </TableHeader>

    <TableBody>
      <TableRow v-for="row in table.getRowModel().rows" :key="row.id">
        <TableCell
          v-for="cell in row.getVisibleCells()"
          :key="cell.id"
          :class="getCellClasses(cell)"
        >
          <FlexRender :props="cell.getContext()" :render="cell.column.columnDef.cell" />
        </TableCell>
      </TableRow>

    </TableBody>
  </Table>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { FlexRender, getCoreRowModel, useVueTable } from '@tanstack/vue-table'
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
  metadata: {
    type: Object,
    default: () => ({}),
  },
})

// Transform backend columns to TanStack format
const transformedColumns = computed(() => {
  const transformColumn = (col, depth = 0) => {
    const baseColumn = {
      id: col.id?.toString() || col.dataField || `col-${Math.random()}`,
      accessorKey: col.dataField,
      header: col.title?.replace(/<br\s*\/?>/gi, '\n') || '',
      meta: {
        width: col.width,
        depth,
      },
      cell: ({ getValue, row }) => {
        const value = getValue()
        return formatCellValue(value, col, row.original)
      },
    }

    // Handle grouped columns
    if (col.children && col.children.length > 0) {
      baseColumn.columns = col.children.map(child => transformColumn(child, depth + 1))
    }

    return baseColumn
  }

  return props.columns.map(col => transformColumn(col))
})

// Create table instance as computed property
const table = computed(() => useVueTable({
  data: props.data,
  columns: transformedColumns.value,
  getCoreRowModel: getCoreRowModel(),
}))

const formatCellValue = (value, columnMeta, rowData) => {
  if (value === null || value === undefined) return ''

  // Apply number formatting
  if (columnMeta.numberFormat !== null && typeof value === 'number') {
    return value.toLocaleString('uz-UZ', {
      minimumFractionDigits: columnMeta.numberFormat || 0,
      maximumFractionDigits: columnMeta.numberFormat || 0,
    })
  }

  return value
}

// Get header classes for styling
const getHeaderClasses = (header, groupIndex) => {
  const classes = ['text-center', 'border-r']
  const meta = header.column.columnDef.meta || {}

  if (meta.fixed === 'left') {
    classes.push('sticky', 'z-20', 'bg-background')
    // Dynamic left positioning based on previous fixed columns
    const leftOffset = getLeftOffset(header.column.id)
    if (leftOffset > 0) {
      classes.push(`left-[${leftOffset}px]`)
    } else {
      classes.push('left-0')
    }
  }

  if (meta.fixed === 'right') {
    classes.push('sticky', 'z-20', 'bg-background', 'border-l')
    const rightOffset = getRightOffset(header.column.id)
    if (rightOffset > 0) {
      classes.push(`right-[${rightOffset}px]`)
    } else {
      classes.push('right-0')
    }
  }

  return classes
}

// Get cell classes for styling
const getCellClasses = (cell) => {
  const classes = ['text-center', 'border-r']
  const meta = cell.column.columnDef.meta || {}

  if (meta.fixed === 'left') {
    classes.push('sticky', 'z-10', 'bg-background')
    const leftOffset = getLeftOffset(cell.column.id)
    if (leftOffset > 0) {
      classes.push(`left-[${leftOffset}px]`)
    } else {
      classes.push('left-0')
    }
  }

  if (meta.fixed === 'right') {
    classes.push('sticky', 'z-10', 'bg-background', 'border-l')
    const rightOffset = getRightOffset(cell.column.id)
    if (rightOffset > 0) {
      classes.push(`right-[${rightOffset}px]`)
    } else {
      classes.push('right-0')
    }
  }

  return classes
}

// Calculate left offset for fixed columns
const getLeftOffset = (columnId) => {
  // This would need to be calculated based on the widths of previous left-fixed columns
  // For now, returning 0 as a placeholder
  return 0
}

// Calculate right offset for fixed columns
const getRightOffset = (columnId) => {
  // This would need to be calculated based on the widths of following right-fixed columns
  // For now, returning 0 as a placeholder
  return 0
}

// Calculate header rowspan
const getHeaderRowSpan = (header, groupIndex) => {
  // Calculate rowspan based on depth and whether column has children
  const totalDepth = table.value.getHeaderGroups().length
  const hasChildren = header.column.columnDef.columns && header.column.columnDef.columns.length > 0

  if (!hasChildren && groupIndex === 0) {
    return totalDepth
  }

  return 1
}
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
  background: linear-gradient(to right, rgba(0, 0, 0, 0.08), transparent);
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
  background: linear-gradient(to left, rgba(0, 0, 0, 0.08), transparent);
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

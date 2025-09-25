<template>
  <div class="relative border rounded-lg overflow-auto max-w-full">
    <Table class="min-w-[1200px] w-full table-fixed">
      <TableHeader>
        <TableRow v-for="headerGroup in table.getHeaderGroups()" :key="headerGroup.id">
          <TableHead
            v-for="(header, headerIndex) in headerGroup.headers"
            :key="header.id"
            :class="getHeaderClasses(header, headerIndex)"
            :colspan="header.colSpan"
            :rowspan="getHeaderRowspan(header)"
            :style="getHeaderStyles(header, headerIndex)"
          >
            <div
              v-html="typeof header.column.columnDef.header === 'function' ? header.column.columnDef.header() : header.column.columnDef.header"
              class="text-xs leading-tight"
            ></div>
          </TableHead>
        </TableRow>
      </TableHeader>

      <TableBody>
        <TableRow v-for="row in table.getRowModel().rows" :key="row.id">
          <TableCell
            v-for="(cell, cellIndex) in row.getVisibleCells()"
            :key="cell.id"
            :class="getCellClasses(cell, cellIndex)"
            :style="getCellStyles(cell, cellIndex)"
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
import { createColumnHelper, FlexRender, getCoreRowModel, useVueTable } from '@tanstack/vue-table'
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
  showSummary: {
    type: Boolean,
    default: false,
  },
})

const columnHelper = createColumnHelper()

const transformedColumns = computed(() => {
  const transformColumn = (col) => {
    if (col.children && col.children.length > 0) {
      return columnHelper.group({
        header: col.title,
        columns: col.children.map((child) => transformColumn(child)),
        width: col.width,
        meta: {
          attributes: col.attributes || {},
          originalColumn: col,
        },
      })
    } else {
      return columnHelper.accessor(col.dataField, {
        header: () => col.title,
        cell: (info) => info.getValue(),
        width: col.width,
        meta: {
          attributes: col.attributes || {},
          originalColumn: col,
        },
      })
    }
  }

  return [...props.columns].map((col) => transformColumn(col))
})

const table = useVueTable({
  get data() {
    return props.data
  },
  get columns() {
    return transformedColumns.value
  },
  getCoreRowModel: getCoreRowModel(),
})

// Get header classes including fixed positioning
const getHeaderClasses = (header, headerIndex) => {
  const classes = ['text-center', 'border-r', 'bg-background']

  const leftFixed = props.leftFixed || 0
  const rightFixed = props.rightFixed || 0
  const totalHeaders = table.getHeaderGroups()[0]?.headers.length || 0

  if (headerIndex < leftFixed) {
    classes.push('sticky', 'z-20', 'border-r-2', 'border-r-gray-300')
  }

  if (rightFixed > 0 && headerIndex >= totalHeaders - rightFixed) {
    classes.push('sticky', 'z-20', 'border-l-2', 'border-l-gray-300')
  }

  return classes
}

// Get header styles for fixed positioning
const getHeaderStyles = (header, headerIndex) => {
  const leftFixed = props.leftFixed || 0
  const rightFixed = props.rightFixed || 0
  const styles = {}
  const headerGroups = table.getHeaderGroups()
  const totalHeaders = headerGroups[0]?.headers.length || 0

  // Set column width
  const width = getColumnWidth(header.column.columnDef)
  styles.width = `${width}px`
  styles.minWidth = `${width}px`
  styles.maxWidth = `${width}px`

  if (headerIndex < leftFixed) {
    // Calculate left offset for sticky positioning
    let leftOffset = 0

    if (headerGroups.length > 0) {
      const currentHeaderGroup = headerGroups[0]
      for (let i = 0; i < headerIndex; i++) {
        const prevHeader = currentHeaderGroup.headers[i]
        const width = getColumnWidth(prevHeader.column.columnDef)
        leftOffset += width
      }
    }

    styles.left = `${leftOffset}px`
    styles.position = 'sticky'
  }

  if (headerIndex >= totalHeaders - rightFixed) {
    // Calculate right offset for sticky positioning
    let rightOffset = 0

    if (headerGroups.length > 0) {
      const currentHeaderGroup = headerGroups[0]
      for (let i = headerIndex + 1; i < totalHeaders; i++) {
        const nextHeader = currentHeaderGroup.headers[i]
        const width = getColumnWidth(nextHeader.column.columnDef)
        rightOffset += width
      }
    }

    styles.right = `${rightOffset}px`
    styles.position = 'sticky'
  }

  return styles
}

// Get cell classes including fixed positioning
const getCellClasses = (cell, cellIndex) => {
  const classes = ['text-center', 'border-r', 'bg-background']
  const leftFixed = props.leftFixed || 0
  const rightFixed = props.rightFixed || 0
  const totalColumns = table.getAllColumns().filter((col) => col.getIsVisible()).length

  if (cellIndex < leftFixed) {
    classes.push('sticky', 'z-10', 'border-r-2', 'border-r-gray-300')
  }

  if (rightFixed > 0 && cellIndex >= totalColumns - rightFixed) {
    classes.push('sticky', 'z-10', 'border-l-2', 'border-l-gray-300')
  }

  return classes
}

// Get cell styles for fixed positioning
const getCellStyles = (cell, cellIndex) => {
  const leftFixed = props.leftFixed || 0
  const rightFixed = props.rightFixed || 0
  const styles = {}
  const visibleColumns = table.getAllColumns().filter((col) => col.getIsVisible())
  const totalColumns = visibleColumns.length

  // Set column width
  const width = getColumnWidth(cell.column.columnDef)
  styles.width = `${width}px`
  styles.minWidth = `${width}px`
  styles.maxWidth = `${width}px`

  if (cellIndex < leftFixed) {
    // Calculate left offset for sticky positioning
    let leftOffset = 0

    for (let i = 0; i < cellIndex; i++) {
      const prevColumn = visibleColumns[i]
      const width = getColumnWidth(prevColumn.columnDef)
      leftOffset += width
    }

    styles.left = `${leftOffset}px`
    styles.position = 'sticky'
  }

  if (cellIndex >= totalColumns - rightFixed) {
    // Calculate right offset for sticky positioning
    let rightOffset = 0

    for (let i = cellIndex + 1; i < totalColumns; i++) {
      const nextColumn = visibleColumns[i]
      const width = getColumnWidth(nextColumn.columnDef)
      rightOffset += width
    }

    styles.right = `${rightOffset}px`
    styles.position = 'sticky'
  }

  return styles
}

// Get header rowspan from column attributes
const getHeaderRowspan = (header) => {
  const attributes = header.column.columnDef.meta?.attributes || {}
  return attributes.rowspan || 1
}

// Get column width from metadata or use default
const getColumnWidth = (columnDef) => {
  // First try to get from meta if available
  if (columnDef.meta?.originalColumn?.width) {
    return parseInt(columnDef.meta.originalColumn.width) || 100
  }

  // Find the original column metadata
  const findColumnMeta = (columns, accessor) => {
    for (const col of columns) {
      if (col.dataField === accessor) {
        return col
      }
      if (col.children && col.children.length > 0) {
        const found = findColumnMeta(col.children, accessor)
        if (found) return found
      }
    }
    return null
  }

  const columnMeta = findColumnMeta(props.columns, columnDef.accessorKey)
  if (columnMeta && columnMeta.width) {
    return parseInt(columnMeta.width) || 100
  }

  // Default widths based on column position
  if (columnDef.accessorKey === '_rownum') return 50
  if (columnDef.accessorKey === 'name') return 350
  return 100
}
</script>

<style scoped>
/* Ensure sticky columns have proper shadows */
:deep(.sticky) {
  box-shadow: 2px 0 4px rgba(0, 0, 0, 0.1);
  background-color: inherit;
  z-index: 10;
}

/* Fix for sticky header cells */
:deep(th.sticky) {
  z-index: 20;
  background-color: hsl(var(--background));
}

/* Fix for sticky body cells */
:deep(td.sticky) {
  z-index: 10;
  background-color: hsl(var(--background));
}

/* Ensure table layout is fixed for consistent column widths */
:deep(.table) {
  table-layout: fixed;
}

/* Hide horizontal scrollbar but keep functionality */
.overflow-auto::-webkit-scrollbar {
  height: 8px;
}

.overflow-auto::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.overflow-auto::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 4px;
}

.overflow-auto::-webkit-scrollbar-thumb:hover {
  background: #a1a1a1;
}
</style>

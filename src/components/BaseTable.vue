<template>
  <div class="relative border rounded-lg overflow-auto max-w-full">
    <Table class="w-full">
      <TableHeader>
        <TableRow v-for="headerGroup in table.getHeaderGroups()" :key="headerGroup.id">
          <TableHead
            v-for="header in headerGroup.headers"
            v-show="shouldShowHeader(header, headerGroup.depth)"
            :key="header.id"
            :class="[
              'text-center border-r bg-background',
              {
                'sticky z-20': isPinnedColumn(header),
                'border-r-2 border-r-gray-300':
                  getColumnPinning(header) === 'left' && isLastPinnedLeft(header),
                'border-l-2 border-l-gray-300':
                  getColumnPinning(header) === 'right' && isFirstPinnedRight(header),
              },
            ]"
            :colspan="header.colSpan"
            :rowspan="getHeaderRowSpan(header)"
            :style="getHeaderStyle(header)"
          >
            <div v-html="header.column.columnDef.header" />
          </TableHead>
        </TableRow>
      </TableHeader>

      <TableBody>
        <TableRow v-for="row in table.getRowModel().rows" :key="row.id">
          <!-- Check if this is a group header row -->
          <template v-if="row.original.__isGroupHeader">
            <TableCell
              :colspan="leafColumns.length"
              class="text-center font-semibold py-3 border-b-2 border-gray-300 bg-gray-100 group-header"
            >
              {{ row.original.__groupName }}
            </TableCell>
          </template>

          <!-- Regular data row -->
          <template v-else>
            <TableCell
              v-for="cell in row.getVisibleCells()"
              :key="cell.id"
              :class="[
                'text-center border-r bg-background',
                {
                  'sticky z-10': cell.column.getIsPinned(),
                  'border-r-2 border-r-gray-300':
                    cell.column.getIsPinned() === 'left' && isLastPinnedLeftCell(cell),
                  'border-l-2 border-l-gray-300':
                    cell.column.getIsPinned() === 'right' && isFirstPinnedRightCell(cell),
                },
              ]"
              :style="getCellStyle(cell)"
            >
              {{ cell.getValue() }}
            </TableCell>
          </template>
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
    required: false,
  },
  leftFixed: {
    type: Number,
    default: 0,
  },
  rightFixed: {
    type: Number,
    default: 0,
  },
  groupingField: {
    type: String,
    default: null,
  },
  tableMetadata: {
    type: Object,
    default: null,
  },
})

// Use tableMetadata if provided, otherwise fall back to individual props
const effectiveLeftFixed = computed(() => {
  return props.tableMetadata?.leftFixed ?? props.leftFixed
})

const effectiveRightFixed = computed(() => {
  return props.tableMetadata?.rightFixed ?? props.rightFixed
})

const effectiveGroupingField = computed(() => {
  return props.tableMetadata?.groupingField ?? props.groupingField
})

const effectiveColumns = computed(() => {
  return props.tableMetadata?.columns ?? props.columns
})

const columnPinning = ref({
  left: [],
  right: [],
})

const transformedColumns = computed(() => {
  const transformColumn = (col, leafIndex, depth = 0) => {
    const colRowSpan = col.attributes?.rowspan || 1 // Default to 1 if not specified

    if (col.children?.length) {
      const transformedChildren = []
      const leafIndices = []
      let currentLeafIndex = leafIndex

      col.children.forEach((child) => {
        const {
          column,
          nextIndex,
          leafIndices: childLeafIndices,
        } = transformColumn(child, currentLeafIndex, depth + 1)
        transformedChildren.push(column)
        leafIndices.push(...childLeafIndices)
        currentLeafIndex = nextIndex
      })

      return {
        column: {
          id: col.id?.toString() || col.dataField || `group_${leafIndex}_${depth}`,
          header: col.title,
          columns: transformedChildren,
          meta: {
            originalColumn: col,
            leafIndices,
            isGroup: true,
            depth,
            rowSpan: colRowSpan, // Add rowSpan to meta
          },
        },
        nextIndex: currentLeafIndex,
        leafIndices,
      }
    }

    return {
      column: {
        id: col.id?.toString() || col.dataField || `col_${leafIndex}`,
        accessorKey: col.dataField,
        header: col.title,
        size: parseInt(col.width) || 150,
        cell: (info) => info.getValue(),
        meta: {
          originalColumn: col,
          leafIndex,
          isGroup: false,
          depth,
          rowSpan: colRowSpan, // Add rowSpan to meta for leaf columns too
        },
      },
      nextIndex: leafIndex + 1,
      leafIndices: [leafIndex],
    }
  }

  let currentLeafIndex = 0
  return effectiveColumns.value.map((col) => {
    const { column, nextIndex } = transformColumn(col, currentLeafIndex)
    currentLeafIndex = nextIndex
    return column
  })
})

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

// Create enhanced data with group information
const enhancedData = computed(() => {
  if (!effectiveGroupingField.value) {
    return props.data.map((item, index) => ({ ...item, __rowIndex: index, __isGroupHeader: false }))
  }

  const result = []
  const groups = {}

  // Group the data
  props.data.forEach((item) => {
    const groupValue = item[effectiveGroupingField.value] || 'Ungrouped'
    if (!groups[groupValue]) {
      groups[groupValue] = []
    }
    groups[groupValue].push(item)
  })

  // Add group headers and data rows
  let rowIndex = 0
  Object.entries(groups).forEach(([groupName, items]) => {
    // Add group header row
    result.push({
      __isGroupHeader: true,
      __groupName: groupName,
      __rowIndex: rowIndex++,
      [effectiveGroupingField.value]: groupName,
    })

    // Add data rows
    items.forEach((item) => {
      result.push({ ...item, __rowIndex: rowIndex++, __isGroupHeader: false })
    })
  })

  return result
})

watch(
  [effectiveLeftFixed, effectiveRightFixed, leafColumns],
  () => {
    const leaves = leafColumns.value

    const leftPinned =
      effectiveLeftFixed.value > 0
        ? leaves.slice(0, effectiveLeftFixed.value).map((col) => col.id)
        : []

    const rightPinned =
      effectiveRightFixed.value > 0
        ? leaves.slice(-effectiveRightFixed.value).map((col) => col.id)
        : []

    columnPinning.value = {
      left: leftPinned,
      right: rightPinned,
    }
  },
  { immediate: true },
)

const table = useVueTable({
  get data() {
    return enhancedData.value
  },
  get columns() {
    return transformedColumns.value
  },
  getCoreRowModel: getCoreRowModel(),
  enableColumnPinning: true,
  state: {
    columnPinning: columnPinning.value,
  },
  enableColumnResizing: true,
  columnResizeMode: 'onChange',
})

const isPinnedColumn = (header) => {
  if (header.column.getIsPinned()) return true

  const meta = header.column.columnDef.meta
  if (meta?.isGroup) {
    const { leafIndices = [] } = meta
    const { left: pinnedLeft, right: pinnedRight } = columnPinning.value
    const leaves = leafColumns.value

    return leafIndices.some((index) => {
      const leaf = leaves[index]
      return leaf && (pinnedLeft.includes(leaf.id) || pinnedRight.includes(leaf.id))
    })
  }

  return false
}

// Helper function to get the pinning side of a column
const getColumnPinning = (header) => {
  const directPinning = header.column.getIsPinned()
  if (directPinning) return directPinning

  const meta = header.column.columnDef.meta
  if (meta?.isGroup) {
    const { leafIndices = [] } = meta
    const { left: pinnedLeft, right: pinnedRight } = columnPinning.value
    const leaves = leafColumns.value

    const hasLeftPinned = leafIndices.some((index) => {
      const leaf = leaves[index]
      return leaf && pinnedLeft.includes(leaf.id)
    })
    if (hasLeftPinned) return 'left'

    const hasRightPinned = leafIndices.some((index) => {
      const leaf = leaves[index]
      return leaf && pinnedRight.includes(leaf.id)
    })
    if (hasRightPinned) return 'right'
  }

  return false
}

// Check if this is the last pinned column on the left
const isLastPinnedLeft = (header) => {
  if (header.column.columnDef.meta?.isGroup) {
    const leafIndices = header.column.columnDef.meta.leafIndices || []
    const leaves = leafColumns.value
    const lastLeftPinnedId = columnPinning.value.left[columnPinning.value.left.length - 1]

    return leafIndices.some((index) => {
      const leaf = leaves[index]
      return leaf && leaf.id === lastLeftPinnedId
    })
  }

  return header.column.id === columnPinning.value.left[columnPinning.value.left.length - 1]
}

// Check if this is the first pinned column on the right
const isFirstPinnedRight = (header) => {
  if (header.column.columnDef.meta?.isGroup) {
    const leafIndices = header.column.columnDef.meta.leafIndices || []
    const leaves = leafColumns.value
    const firstRightPinnedId = columnPinning.value.right[0]

    return leafIndices.some((index) => {
      const leaf = leaves[index]
      return leaf && leaf.id === firstRightPinnedId
    })
  }

  return header.column.id === columnPinning.value.right[0]
}

// Check if this cell is the last pinned on the left
const isLastPinnedLeftCell = (cell) => {
  return cell.column.id === columnPinning.value.left[columnPinning.value.left.length - 1]
}

// Check if this cell is the first pinned on the right
const isFirstPinnedRightCell = (cell) => {
  return cell.column.id === columnPinning.value.right[0]
}

// Calculate styles for headers (both pinned and grouped)
const getHeaderStyle = (header) => {
  const pinning = getColumnPinning(header)
  if (!pinning) return {}

  const style = {}

  if (pinning === 'left') {
    // For left pinning, calculate position
    let leftPos = 0
    const leaves = leafColumns.value
    const pinnedLeft = columnPinning.value.left

    if (header.column.columnDef.meta?.isGroup) {
      // For group headers, find the position of the first leaf
      const leafIndices = header.column.columnDef.meta.leafIndices || []
      const firstLeafIndex = Math.min(...leafIndices)

      // Calculate position based on preceding pinned columns
      for (let i = 0; i < firstLeafIndex; i++) {
        const leaf = leaves[i]
        if (leaf && pinnedLeft.includes(leaf.id)) {
          leftPos += leaf.size || 150
        }
      }
    } else {
      // For leaf columns, use the built-in method
      leftPos = header.column.getStart('left')
    }

    style.left = `${leftPos}px`
  } else if (pinning === 'right') {
    // For right pinning
    let rightPos = 0
    const leaves = leafColumns.value
    const pinnedRight = columnPinning.value.right

    if (header.column.columnDef.meta?.isGroup) {
      // For group headers, calculate from the right
      const leafIndices = header.column.columnDef.meta.leafIndices || []
      const lastLeafIndex = Math.max(...leafIndices)

      // Calculate position based on succeeding pinned columns
      for (let i = leaves.length - 1; i > lastLeafIndex; i--) {
        const leaf = leaves[i]
        if (leaf && pinnedRight.includes(leaf.id)) {
          rightPos += leaf.size || 150
        }
      }
    } else {
      // For leaf columns, use the built-in method
      rightPos = header.column.getAfter('right')
    }

    style.right = `${rightPos}px`
  }

  // Calculate width for group headers
  if (header.column.columnDef.meta?.isGroup) {
    const leafIndices = header.column.columnDef.meta.leafIndices || []
    const leaves = leafColumns.value
    let totalWidth = 0

    leafIndices.forEach((index) => {
      const leaf = leaves[index]
      if (leaf) {
        totalWidth += leaf.size || 150
      }
    })

    style.width = `${totalWidth}px`
    style.minWidth = `${totalWidth}px`
  } else {
    const width = header.column.columnDef.size || header.column.getSize()
    style.width = `${width}px`
    style.minWidth = `${width}px`
  }

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

function shouldShowHeader(header, depth) {
  const originalColumn = header.column.columnDef.meta?.originalColumn
  if (!originalColumn) return true

  const hasRowspan = originalColumn.attributes?.rowspan === 2
  return !(hasRowspan && depth > 0)
}

function getHeaderRowSpan(header) {
  const originalColumn = header.column.columnDef.meta?.originalColumn
  if (!originalColumn) return 1

  return originalColumn.attributes?.rowspan || 1
}
</script>

<style scoped>
/* Ensure table has proper layout */
table {
  table-layout: fixed;
}

/* Ensure proper z-index layering */
thead th.sticky {
  z-index: 20 !important;
}

tbody td.sticky {
  z-index: 10 !important;
}

/* Group header styling */
.bg-gray-100 {
  background-color: #f3f4f6 !important;
}

/* Simple group header styles */
td[colspan] {
  text-align: center !important;
}
.group-header {
  position: sticky !important;
  left: 50% !important;
  transform: translateX(-50%) !important;
  display: inline-flex !important;
  align-items: center !important;
  justify-content: center !important;
  z-index: 15 !important;
  min-width: 100% !important;
  background-color: white !important;
}
</style>

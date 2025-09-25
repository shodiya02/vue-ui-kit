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
                'sticky z-20': isPinnedColumn(header),
                'border-r-2 border-r-gray-300': getColumnPinning(header) === 'left' && isLastPinnedLeft(header),
                'border-l-2 border-l-gray-300': getColumnPinning(header) === 'right' && isFirstPinnedRight(header),
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
                'border-r-2 border-r-gray-300': cell.column.getIsPinned() === 'left' && isLastPinnedLeftCell(cell),
                'border-l-2 border-l-gray-300': cell.column.getIsPinned() === 'right' && isFirstPinnedRightCell(cell),
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
  const transformColumn = (col, leafIndex, depth = 0) => {
    // For grouped columns (have children)
    if (col.children && col.children.length > 0) {
      let currentLeafIndex = leafIndex
      const transformedChildren = []
      const leafIndices = []

      col.children.forEach((child) => {
        const result = transformColumn(child, currentLeafIndex, depth + 1)
        transformedChildren.push(result.column)
        leafIndices.push(...result.leafIndices)
        currentLeafIndex = result.nextIndex
      })

      return {
        column: {
          id: col.id?.toString() || col.dataField || `group_${leafIndex}_${depth}`,
          header: col.title,
          columns: transformedChildren,
          meta: {
            originalColumn: col,
            leafIndices: leafIndices, // Store all leaf indices under this group
            isGroup: true,
            depth: depth,
          },
        },
        nextIndex: currentLeafIndex,
        leafIndices: leafIndices,
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
          leafIndex: leafIndex,
          isGroup: false,
          depth: depth,
        },
      },
      nextIndex: leafIndex + 1,
      leafIndices: [leafIndex],
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
    const leftPinned = props.leftFixed > 0
      ? leaves.slice(0, props.leftFixed).map((col) => col.id)
      : []

    // Set right pinned columns
    const rightPinned = props.rightFixed > 0
      ? leaves.slice(-props.rightFixed).map((col) => col.id)
      : []

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

// Helper function to determine if a header should be pinned
const isPinnedColumn = (header) => {
  // If the column itself is pinned
  if (header.column.getIsPinned()) return true

  // If it's a group column, check if any of its leaves are pinned
  if (header.column.columnDef.meta?.isGroup) {
    const leafIndices = header.column.columnDef.meta.leafIndices || []
    const pinnedLeft = columnPinning.value.left
    const pinnedRight = columnPinning.value.right
    const leaves = leafColumns.value

    return leafIndices.some(index => {
      const leaf = leaves[index]
      return leaf && (pinnedLeft.includes(leaf.id) || pinnedRight.includes(leaf.id))
    })
  }

  return false
}

// Helper function to get the pinning side of a column
const getColumnPinning = (header) => {
  // If the column itself is pinned
  const directPinning = header.column.getIsPinned()
  if (directPinning) return directPinning

  // If it's a group column, determine pinning based on its leaves
  if (header.column.columnDef.meta?.isGroup) {
    const leafIndices = header.column.columnDef.meta.leafIndices || []
    const pinnedLeft = columnPinning.value.left
    const pinnedRight = columnPinning.value.right
    const leaves = leafColumns.value

    const hasLeftPinned = leafIndices.some(index => {
      const leaf = leaves[index]
      return leaf && pinnedLeft.includes(leaf.id)
    })

    const hasRightPinned = leafIndices.some(index => {
      const leaf = leaves[index]
      return leaf && pinnedRight.includes(leaf.id)
    })

    if (hasLeftPinned) return 'left'
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

    return leafIndices.some(index => {
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

    return leafIndices.some(index => {
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

    leafIndices.forEach(index => {
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

/* Shadow for left pinned columns - only on the rightmost edge */
th.border-r-2.border-r-gray-300.sticky,
td.border-r-2.border-r-gray-300.sticky {
  box-shadow: 2px 0 5px -2px rgba(0, 0, 0, 0.15);
}

/* Shadow for right pinned columns - only on the leftmost edge */
th.border-l-2.border-l-gray-300.sticky,
td.border-l-2.border-l-gray-300.sticky {
  box-shadow: -2px 0 5px -2px rgba(0, 0, 0, 0.15);
}

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
</style>

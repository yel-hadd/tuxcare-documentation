<template>
  <div class="cve-tracker">
    <!-- Loading State -->
    <div v-if="loading" class="loading-container">
      <div class="loading-spinner"></div>
      <p class="loading-text">Loading resolved CVEs...</p>
    </div>
    <!-- Error State -->
    <div v-else-if="error" class="error-container">
      <h5 class="error-title">Error Loading CVE Data</h5>
      <p class="error-message">{{ error }}</p>
      <button class="retry-button" @click="fetchCVEData">
        Retry
      </button>
    </div>
    <!-- Success State -->
    <div v-else>
      <!-- Stats Cards -->
      <div class="stats-grid">
        <div class="stat-card stat-total">
          <div class="stat-number">{{ cveData.total }}</div>
          <div class="stat-label">Total Resolved</div>
        </div>
        <div class="stat-card stat-critical">
          <div class="stat-number">{{ stats.critical + stats.high }}</div>
          <div class="stat-label">Critical + High</div>
        </div>
        <div class="stat-card stat-medium">
          <div class="stat-number">{{ stats.medium }}</div>
          <div class="stat-label">Medium</div>
        </div>
        <div class="stat-card stat-low">
          <div class="stat-number">{{ stats.low }}</div>
          <div class="stat-label">Low</div>
        </div>
      </div>
      <!-- Table Container -->
      <div class="table-container">
        <div class="table-wrapper">
          <div ref="tableContainer">
            <table ref="cveTable" class="cve-table">
              <thead>
                <tr>
                  <th class="col-cve">CVE Name</th>
                  <th class="col-severity">Severity</th>
                  <th class="col-score">Score</th> <!-- This will be hidden -->
                  <th class="col-group">Group</th>
                  <th class="col-package">Package</th>
                  <th class="col-vulnerable">Vulnerable Ver.</th>
                  <th class="col-fixed">Fixed Ver.</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="cve in cveData.data" :key="cve.cve_name">
                  <td class="col-cve">
                    <code class="cve-code">{{ cve.cve_name }}</code>
                  </td>
                  <td class="col-severity">
                    <span :class="getSeverityClass(cve.severity_name)" class="severity-badge">
                      {{ cve.severity_name || 'Unknown' }}
                    </span>
                  </td>
                  <td class="col-score">
                    <span class="score-value">{{ cve.severity_score !== null ? cve.severity_score.toFixed(1) : 'N/A' }}</span>
                  </td> <!-- This will be hidden -->
                  <td class="col-group">
                    <span class="group-text">{{ cve.group }}</span>
                  </td>
                  <td class="col-package">
                    <span class="package-name">{{ cve.name }}</span>
                  </td>
                  <td class="col-vulnerable">
                    <code class="version-vulnerable">{{ cve.version }}</code>
                  </td>
                  <td class="col-fixed">
                    <code class="version-fixed">{{ cve.fixed_version }}</code>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CVETracker',
  props: {
    apiUrl: {
      type: String,
      default: 'https://spring-els-cves.cl-edu.com/api/v1/resolved-cves'
    }
  },
  data() {
    return {
      loading: true,
      error: null,
      cveData: {
        total: 0,
        data: []
      },
      dataTable: null
    }
  },
  computed: {
    stats() {
      const stats = { critical: 0, high: 0, medium: 0, low: 0, unknown: 0 }
      this.cveData.data.forEach(item => {
        const severity = (item.severity_name || 'unknown').toLowerCase()
        if (stats.hasOwnProperty(severity)) {
          stats[severity]++
        } else {
          stats.unknown++
        }
      })
      return stats
    }
  },
  mounted() {
    this.fetchCVEData()
  },
  beforeDestroy() {
    if (this.dataTable) {
      this.dataTable.destroy()
    }
  },
  methods: {
    async fetchCVEData() {
      this.loading = true
      this.error = null
      try {
        const response = await fetch(this.apiUrl)
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`)
        }
        this.cveData = await response.json()
        this.loading = false
        this.$nextTick(() => {
          this.waitForjQueryAndInitialize()
        })
      } catch (err) {
        this.error = err.message
        this.loading = false
      }
    },
    waitForjQueryAndInitialize() {
      const checkjQuery = () => {
        if (window.jQuery && window.jQuery.fn && window.jQuery.fn.DataTable) {
          this.initializeDataTable()
        } else {
          // Retry after 1 second
          setTimeout(checkjQuery, 1000)
        }
      }
      checkjQuery()
    },
    getSeverityClass(severity) {
      const severityLower = (severity || 'unknown').toLowerCase()
      return `severity-${severityLower}`
    },
    initializeDataTable() {
      // Skip DataTables initialization on mobile (screen width < 768px)
      if (window.innerWidth < 768) {
        return
      }

      // Custom sorting for severity column
      window.jQuery.fn.dataTable.ext.type.detect.unshift(function(data) {
        return data && typeof data === 'string' && data.includes('severity-') ? 'severity-sort' : null
      })
      window.jQuery.fn.dataTable.ext.type.order['severity-sort-pre'] = function(data) {
        const severity = data.replace(/<[^>]*>/g, '').toLowerCase().trim()
        const severityOrder = {
          'critical': 4,
          'high': 3,
          'medium': 2,
          'low': 1,
          'unknown': 0
        }
        return severityOrder[severity] !== undefined ? severityOrder[severity] : 0
      }

      // Destroy existing DataTable if it exists
      if (this.dataTable) {
        this.dataTable.destroy()
      }

      // Initialize DataTable
      this.dataTable = window.jQuery(this.$refs.cveTable).DataTable({
        order: [[1, 'desc'], [2, 'desc']], // Sort by severity, then score
        pageLength: 10,
        lengthMenu: [[10, 25, 50, 100, -1], [10, 25, 50, 100, "All"]],
        responsive: false,
        scrollX: false,
        autoWidth: true,
        columnDefs: [
          {
            targets: '_all',
            render: function(data, type, row) {
              return type === 'display' ? data : data.replace(/<[^>]+>/g, '')
            }
          },
          {
            targets: [5, 6], // Vulnerable Ver. and Fixed Ver.
            className: 'no-wrap',
            visible: true
          },
          {
            targets: 1, // Severity column
            type: 'severity-sort',
            orderable: true
          },
          {
            targets: 2, // Score column
            visible: false
          }
        ],
        language: {
          search: "Search:",
          lengthMenu: "Show _MENU_ entries",
          info: "Showing _START_ to _END_ of _TOTAL_ CVEs",
          emptyTable: "No resolved CVEs found"
        },
        dom: '<"table-controls"lf>rt<"table-footer"ip>'
      })
    }
  }
}
</script>

<style lang="stylus" scoped>

.cve-tracker
  background-color white
  padding 1rem
  border none
  border-radius 0
  font-family -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif
  width 98%
  max-width 98%

// Hide Score column
.cve-table
  th.col-score
    display none
  td.col-score
    display none

.cve-tracker
  background-color white
  padding 1rem
  border none
  border-radius 0
  font-family -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif
  width 98%
  max-width 98%

// Loading States
.loading-container
  display flex
  flex-direction column
  align-items center
  justify-content center
  padding 3rem 0
.loading-spinner
  width 3rem
  height 3rem
  border 3px solid #e3e3e3
  border-top 3px solid #3498db
  border-radius 50%
  animation spin 1s linear infinite
@keyframes spin
  0%
    transform rotate(0deg)
  100%
    transform rotate(360deg)
.loading-text
  margin-top 1rem
  color #666
  font-size 0.9rem

// Error States
.error-container
  background-color #fef2f2
  border 1px solid #fecaca
  color #dc2626
  padding 1rem
  border-radius 8px
.error-title
  font-weight 600
  margin-bottom 0.5rem
.error-message
  margin-bottom 0.75rem
.retry-button
  background-color #3b82f6
  color white
  padding 0.5rem 1rem
  border none
  border-radius 6px
  font-size 0.875rem
  cursor pointer
  transition background-color 0.2s
  &:hover
    background-color #2563eb

// Stats Grid
.stats-grid
  display grid
  grid-template-columns repeat(2, 1fr)
  gap 1rem
  margin-bottom 1.5rem
  @media (min-width: 768px)
    grid-template-columns repeat(4, 1fr)
.stat-card
  padding 1rem
  border-radius 8px
  color white
  text-align center
.stat-total
  background-color #2563eb
.stat-critical
  background-color #dc2626
.stat-medium
  background-color #d97706
.stat-low
  background-color #16a34a
.stat-number
  font-size 1.5rem
  font-weight bold
  line-height 1
.stat-label
  font-size 0.75rem
  opacity 0.9
  margin-top 0.25rem

// Table Styles
.table-container
  background-color white
  border-radius 8px
  box-shadow 0 1px 3px rgba(0, 0, 0, 0.1)
  border 1px solid #e5e7eb
  width 100%
  max-width 100%
.table-wrapper
  width 100%
  overflow-x hidden
  -webkit-overflow-scrolling touch
.cve-table
  width 100%
  font-size 0.8rem
  border-collapse collapse
  table-layout auto
  thead th
    background-color #0d1e30
    color white
    padding 0.5rem 0.75rem
    text-align left
    font-weight 600
    font-size 0.75rem
    white-space normal
    word-break break-word
    vertical-align top
  tbody td
    padding 0.5rem 0.75rem
    border-bottom 1px solid #e5e7eb
    font-size 0.75rem
    vertical-align top
    word-break break-word
    white-space normal
  tbody tr:hover
    background-color #f9fafb

// Content styling
.cve-code
  background-color #f3f4f6
  padding 0.125rem 0.375rem
  border-radius 3px
  font-size 0.65rem
  font-family 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace
  display inline-block
  max-width 100%
  overflow hidden
  text-overflow ellipsis
  white-space nowrap
.version-vulnerable
  background-color #fef2f2
  color #dc2626
  padding 0.125rem 0.375rem
  border-radius 3px
  font-size 0.65rem
  font-family 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace
  display inline-block
  max-width 100%
  overflow hidden
  text-overflow ellipsis
  white-space nowrap
.version-fixed
  background-color #f0fdf4
  color #16a34a
  padding 0.125rem 0.375rem
  border-radius 3px
  font-size 0.65rem
  font-family 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace
  display inline-block
  max-width 100%
  overflow hidden
  text-overflow ellipsis
  white-space nowrap
.package-name
  font-weight 500
  color #111827
  font-size 0.75rem
  word-wrap break-word
  line-height 1.3
  display block
.group-text
  font-size 0.75rem
  word-wrap break-word
  line-height 1.3
  display block

// Severity badges
.severity-badge
  display inline-flex
  align-items center
  padding 0.125rem 0.5rem
  border-radius 12px
  font-size 0.65rem
  font-weight 600
  text-transform uppercase
  letter-spacing 0.025em
  white-space nowrap
.severity-critical
  background-color #dc2626
  color white
.severity-high
  background-color #ea580c
  color white
.severity-medium
  background-color #ca8a04
  color white
.severity-low
  background-color #16a34a
  color white
.severity-unknown
  background-color #6b7280
  color white

// Mobile optimizations
@media (max-width: 768px)
  .cve-table
    font-size 0.7rem
    thead th
      padding 0.375rem 0.5rem
      font-size 0.7rem
    tbody td
      padding 0.375rem 0.5rem
      font-size 0.7rem
@media (max-width: 767px)
  .table-wrapper
    overflow-x auto

// DataTables custom styling
.cve-tracker :deep(.table-controls)
  display flex
  flex-direction column
  gap 1rem
  margin-bottom 1rem
  @media (min-width: 768px)
    flex-direction row
    align-items center
    justify-content space-between
.cve-tracker :deep(.table-footer)
  display flex
  flex-direction column
  gap 1rem
  margin-top 1rem
  @media (min-width: 768px)
    flex-direction row
    align-items center
    justify-content space-between
.cve-tracker :deep(.dataTables_length),
.cve-tracker :deep(.dataTables_filter),
.cve-tracker :deep(.dataTables_info),
.cve-tracker :deep(.dataTables_paginate)
  color #374151
  font-size 0.875rem
.cve-tracker :deep(.dataTables_filter input)
  border 1px solid #d1d5db
  border-radius 6px
  padding 0.5rem 0.75rem
  font-size 0.875rem
  &:focus
    outline none
    border-color #3b82f6
    box-shadow 0 0 0 3px rgba(59, 130, 246, 0.1)
.cve-tracker :deep(.dataTables_length select)
  border 1px solid #d1d5db
  border-radius 6px
  padding 0.5rem 0.75rem
  font-size 0.875rem
  &:focus
    outline none
    border-color #3b82f6
    box-shadow 0 0 0 3px rgba(59, 130, 246, 0.1)
.cve-tracker :deep(.dataTables_paginate .paginate_button)
  padding 0.5rem 0.75rem
  font-size 0.875rem
  border 1px solid #d1d5db
  border-radius 6px
  margin 0 0.125rem
  text-decoration none
  color #374151
  &:hover
    background-color #f9fafb
    text-decoration none
.cve-tracker :deep(.dataTables_paginate .paginate_button.current)
  background-color #3b82f6
  color white
  border-color #3b82f6
  &:hover
    background-color #2563eb
    border-color #2563eb
</style>
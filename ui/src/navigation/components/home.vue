<template>
  <div
    v-if="widthLessThan('sm')"
    class="mx-2 mx-md-3"
  >
    <div class="d-flex justify-content-between align-items-center">
      <h1 class="mt-3 mb-2">
        {{ i18n['hello_admin'] }}
      </h1>
      <VButton
        v-if="$can('manage', 'all') && $route.name === 'home'"
        :icon="isEdit ? 'md-close' : 'md-create'"
        class="fs-1 bg-transparent"
        type="text"
        @click="isEdit = !isEdit"
      />
    </div>
    <div
      v-if="$route.name === 'home'"
      class="row"
    >
      <LinksSection />
    </div>
  </div>
  <div class="mx-2 mx-md-3 position-relative">
    <h1
      v-if="widthLessThan('sm') && $route.name === 'home'"
      class="mt-3 mb-2"
    >
      {{ dashboardId ? dashboard?.title : i18n['resources'] }}
    </h1>
    <div
      v-else-if="!widthLessThan('sm')"
      class="d-flex justify-content-between align-items-center"
    >
      <h1
        class="mt-3 mb-2"
      >
        {{ i18n['hello_admin'] }}
      </h1>
      <VButton
        v-if="$can('manage', 'all') && $route.name === 'home'"
        :icon="isEdit ? 'md-close' : 'md-create'"
        class="fs-1 bg-transparent"
        type="text"
        @click="isEdit = !isEdit"
      />
    </div>
    <Spin
      v-if="isLoading"
      fix
    />
    <div
      v-if="isEdit"
      style="max-width: 450px; margin: auto;"
    >
      <DashboardSelect v-model="selectedDashboardId" />
      <div class="text-center">
        <VButton
          icon="md-close"
          type="text"
          class="bg-transparent"
          @click="isEdit = false"
        >
          {{ i18n.cancel }}
        </VButton>
        <VButton
          icon="md-checkmark"
          class="bg-transparent"
          type="text"
          @click="selectDashboard"
        >
          {{ i18n.submit }}
        </VButton>
      </div>
    </div>

    <DashboardLayout
      v-else-if="dashboardId && dashboard && $route.name === 'home'"
      :dashboard="dashboard"
    />
    <div
      v-else
      class="row"
    >
      <div
        v-for="resource in visibleResources"
        :key="resource.slug"
        class="col-12 col-md-6 col-lg-4 col-xl-3"
      >
        <RouterLink
          class="ivu-card ivu-card-bordered my-2"
          :to="{ name: 'resources', params: { fragments: [resource.slug] }}"
        >
          <div class="ivu-card-body">
            <p class="fs-4 fw-bold text-dark">
              {{ resource.display_name.replace('/', '\u200B/') }}
            </p>
          </div>
        </RouterLink>
      </div>
      <div
        v-if="!schema.length && databaseSettingsPath"
        class="text-center mb-4"
      >
        <VButton
          icon="md-add"
          size="large"
          :to="databaseSettingsPath"
        >
          {{ i18n.add_database }}
        </VButton>
      </div>
    </div>
    <div
      v-if="!isEdit"
      class="text-center"
      style="bottom: 0"
    >
      <a
        :href="isStandalone ? 'https://github.com/motor-admin/motor-admin' : 'https://github.com/motor-admin/motor-admin-rails'"
        target="_blank"
      >Motor Admin v{{ version }}</a>
    </div>
  </div>
</template>

<script>
import api from 'api'
import LinksSection from 'navigation/components/links_section'
import DashboardSelect from 'dashboards/components/select'
import DashboardLayout from 'dashboards/components/layout'
import { widthLessThan } from 'utils/scripts/dimensions'
import { schema } from 'data_resources/scripts/schema'
import { homepageStore } from '../scripts/homepage_store'
import { version, isStandalone, adminSettingsPath } from 'utils/scripts/configs'

export default {
  name: 'Home',
  components: {
    LinksSection,
    DashboardSelect,
    DashboardLayout
  },
  data () {
    return {
      isEdit: false,
      isLoading: false,
      selectedDashboardId: null,
      dashboard: null
    }
  },
  computed: {
    version: () => version,
    schema: () => schema,
    isStandalone: () => isStandalone,
    homepageStore () {
      return homepageStore
    },
    dashboardId () {
      return homepageStore[0]?.id
    },
    databaseSettingsPath () {
      return adminSettingsPath?.replace('general', 'database')
    },
    visibleResources () {
      return schema.filter((model) => model.visible)
    }
  },
  watch: {
    dashboardId () {
      if (this.dashboardId) {
        this.loadDashboard()
      } else {
        this.dashboard = null
      }
    }
  },
  created () {
    this.selectedDashboardId = homepageStore[0]?.id

    if (this.dashboardId) {
      this.loadDashboard()
    }
  },
  methods: {
    widthLessThan,
    loadDashboard () {
      this.isLoading = true

      return api.get(`dashboards/${this.dashboardId}`, {
        params: {
          include: 'tags,queries',
          fields: {
            queries: 'id,name,preferences'
          }
        }
      }).then((result) => {
        this.dashboard = result.data.data
      }).catch((error) => {
        if (error.response?.data?.errors) {
          this.$Message.error(error.response.data.errors.join('\n'))
        } else {
          this.$Message.error(error.message)
        }
      }).finally(() => {
        this.isLoading = false
      })
    },
    selectDashboard () {
      this.isLoading = true

      api.post('configs', {
        key: 'homepage.layout',
        value: this.selectedDashboardId ? [{ type: 'dashboard', id: this.selectedDashboardId }] : []
      }).then((result) => {
        homepageStore.splice(0, homepageStore.length, ...result.data.data.value)

        if (this.dashboardId) {
          this.loadDashboard().finally(() => {
            this.isEdit = false
          })
        } else {
          this.isEdit = false
          this.isLoading = false
        }
      }).catch((error) => {
        console.error(error)

        this.$Message.error(this.i18n.unable_to_submit_form)
        this.isLoading = false
      })
    }
  }
}
</script>

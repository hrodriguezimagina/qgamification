<template>
  <div id="gamificationCategoryViewIndex">
    <div :id="`gamificationCategory-${category? category.systemName : ''}`">
      <component v-if="activities.length" :is="viewComponent" v-bind="viewProps"/>
    </div>
  </div>
</template>
<script>
export default {
  props: {
    systemName: {default: false},
    view: {default: 'card'},
    btnProps: {
      type: Object, default: () => {
        return {}
      }
    }
  },
  components: {},
  watch: {},
  mounted() {
    this.$nextTick(function () {
      this.init()
    })
  },
  data() {
    return {
      loading: false,
      category: null,
      activities: []
    }
  },
  computed: {
    //Instance the view component to render
    viewComponent() {
      const viewComponentName = this.category.options?.categoryView || "card"
      //Instance default component
      let component = () => import('@imagina/qgamification/_components/categoryView/card.vue')
      //Switch the ciew component type
      switch (viewComponentName) {
        case 'button':
          component = () => import('@imagina/qgamification/_components/categoryView/button.vue')
          break;
        case 'popup':
          component = () => import('@imagina/qgamification/_components/categoryView/popup.vue')
          break;
      }
      //Response
      return component
    },
    //Instance the activity view component
    activityViewComponent() {
      const viewComponentName = this.category.options?.activityView || "listButton"
      return () => import(`@imagina/qgamification/_components/activityView/${viewComponentName}.vue`)
    },
    //Instance the props for the component
    viewProps() {
      return {
        ...this.$props,
        category: this.category,
        activities: this.activities,
        activityViewComponent: this.activityViewComponent
      }
    },
    //Return the necessary settings for this component
    settings() {
      return {
        hcStatus: parseInt(this.$store.getters['qsiteApp/getSettingValueByName']('igamification::hcStatus') || 0),
        hcInternalCategory: this.$store.getters['qsiteApp/getSettingValueByName']('igamification::hcInternalCategory'),
        hcCentralizedSource: this.$store.getters['qsiteApp/getSettingValueByName']('igamification::hcCentralizedSource'),
      }
    },
    //Validate if the category is enable and return it
    categorySystemName() {
      let categorySystemName = this.$clone(this.systemName)
      //Validate and swith the category
      switch (this.systemName) {
        case 'help_center':
          if (!this.settings.hcStatus) categorySystemName = null
          else if (this.settings.hcInternalCategory) categorySystemName = this.settings.hcInternalCategory
          break
      }
      //Response
      return categorySystemName
    },
    //Return the baseUrl to request the activites information
    baseUrl() {
      //Instance the default baseUrl
      let baseUrl = this.$axios.defaults.baseURL
      //Switch the base URL based on systemName
      switch (this.systemName) {
        case 'help_center':
          if (!this.settings.hcInternalCategory && this.settings.hcCentralizedSource)
            baseUrl = `${this.settings.hcCentralizedSource}/api`
          break
      }
      //Response
      return baseUrl
    }
  },
  methods: {
    init() {
      this.getData(true)
    },
    //get data
    async getData(refresh = false) {
      this.loading = true
      if (this.categorySystemName) {
        await this.getCategory(refresh)
        await this.getActivities(refresh)
      }
      this.$emit('loaded')
      this.loading = false
    },
    //Get data
    getCategory(refresh = false) {
      return new Promise((resolve, reject) => {
        if (!this.categorySystemName) return resolve(false)
        //Request params to category
        let requestParams = {
          include: 'fields',
          filter: {field: 'system_name'}
        }
        //Get category
        this.$axios.get(
            `${this.baseUrl}${config('apiRoutes.qgamification.categories')}/${this.categorySystemName}`,
            {params: requestParams}
        ).then(response => {
          this.category = {
            ...response.data.data,
            icon: response.data.data.options?.icon || 'fas fa-gamepad'
          }
          resolve(response.data)
        }).catch(error => {
          resolve([])
        })
      })
    },
    //Get activities
    getActivities(refresh = false) {
      return new Promise(resolve => {
        if (!this.category) return resolve([])
        //Requets params
        let requestParams = {
          include: 'users',
          filter: {categoryId: this.category.id, validateComplete: true, order: {field: 'position', way: 'asc'}}
        }
        //Request
        this.$axios.get(
            `${this.baseUrl}${config('apiRoutes.qgamification.activities')}`,
            {params: requestParams}
        ).then(response => {
          //Get userRoles
          let userRoles = this.$store.state.quserAuth.userData.roles.map(role => role.id)
          //Set only activities without complete
          this.activities = response.data.data.filter(item => {
            if (!item.userCompleted) {
              //Get role id from activity
              let activityRoles = (item.options ? item.options.roles : []).map(item => parseInt(item))
              //Check if current user has access to activity
              let hasAccess = !activityRoles.length ? true : false
              userRoles.forEach(userRole => {
                if (activityRoles.includes(userRole)) hasAccess = true
              })
              //Validate activity permission if is neeed
              if(hasAccess && item.options?.permission) hasAccess = this.$auth.hasAccess(item.options.permission)
              //Response
              if (hasAccess) return item
            }
          }).map(act => ({
            ...act,
            icon: act.options?.icon || 'fa-light fa-check-to-slot'
          }))
          resolve(response.data.data)
        }).catch(error => {
          resolve([])
        })
      })
    }
  }
}
</script>
<style lang="stylus">
</style>

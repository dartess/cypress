<template>
  <div
    class="h-screen min-w-728px grid grid-cols-[auto,1fr]"
    :class="{
      'grid-rows-[64px,1fr]': showHeader
    }"
  >
    <SidebarNavigation
      v-if="renderSidebar"
      class="row-span-full"
    />

    <HeaderBar
      v-if="showHeader"
      :show-browsers="true"
      :page-name="currentRoute.name?.toString()"
      data-cy="app-header-bar"
      :allow-automatic-prompt-open="true"
      @connect-project="showConnectDialog = true"
    />
    <div
      v-if="query.data.value?.baseError || query.data.value?.currentProject?.isLoadingConfigFile || query.data.value?.currentProject?.isLoadingNodeEvents"
      class="bg-white h-full w-full pt-100px top-0 right-0 left-0 z-10 absolute overflow-scroll"
    >
      <BaseError
        v-if="query.data.value?.baseError"
        :gql="query.data.value.baseError"
        @retry="resetErrorAndLoadConfig"
      />
      <div v-else>
        <Spinner />
      </div>
    </div>
    <main
      aria-labelledby="primary-heading"
      class="overflow-auto"
    >
      <router-view v-slot="{ Component, route }">
        <h1
          id="primary-heading"
          class="sr-only"
        >
          {{ route.name }}
        </h1>
        <transition
          name="fade"
          mode="out-in"
        >
          <component :is="Component" />
        </transition>
      </router-view>
      <CloudConnectModals
        v-if="showConnectDialog && query.data.value"
        :show="showConnectDialog"
        :gql="query.data.value"
        @cancel="showConnectDialog = false"
        @success="showConnectDialog = false"
      />
    </main>
  </div>
</template>

<script lang="ts" setup>
import { gql, useQuery, useMutation } from '@urql/vue'
import SidebarNavigation from '../navigation/SidebarNavigation.vue'
import HeaderBar from '@cy/gql-components/HeaderBar.vue'
import BaseError from '@cy/gql-components/error/BaseError.vue'
import Spinner from '@cy/components/Spinner.vue'
import CloudConnectModals from '../runs/modals/CloudConnectModals.vue'

import { useRoute } from 'vue-router'
import { computed, ref } from 'vue'

import { MainAppQueryDocument, MainApp_ResetErrorsAndLoadConfigDocument } from '../generated/graphql'

gql`
fragment MainAppQueryData on Query {
    baseError {
      id
      ...BaseError
    }
    currentProject {
      id
      isLoadingConfigFile
      isLoadingNodeEvents
    }
    ...CloudConnectModals
}
`

gql`
query MainAppQuery {
  ...MainAppQueryData
}
`

gql`
mutation MainApp_ResetErrorsAndLoadConfig($id: ID!) {
  resetErrorAndLoadConfig(id: $id) {
    ...MainAppQueryData
  }
}
`

const currentRoute = useRoute()

const showHeader = computed(() => {
  return currentRoute.meta.header !== false
})

const query = useQuery({
  query: MainAppQueryDocument,
  pause: !showHeader.value,
})
const mutation = useMutation(MainApp_ResetErrorsAndLoadConfigDocument)

const resetErrorAndLoadConfig = (id: string) => {
  if (!mutation.fetching.value) {
    mutation.executeMutation({ id })
  }
}

const renderSidebar = window.__CYPRESS_MODE__ !== 'run'
const showConnectDialog = ref(false)

</script>

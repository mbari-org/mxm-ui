<script setup lang="ts">
import UtlBreadcrumbs from 'components/utl/UtlBreadcrumbs.vue'
import QDarkMode from 'components/utl/QDarkMode.vue'
</script>

<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar class="glossy row justify-between q-pb-sm">
        <q-img
          v-if="$q.screen.gt.sm"
          src="img/new_mbari_logo.png"
          width="54px"
          class="q-mr-lg"
        />

        <q-toolbar-title class="row q-gutter-x-xs" style="font-size: 1.1em">
          MXM
          <div v-if="$q.screen.gt.sm">
            - Mission Execution Mediation Service
          </div>
        </q-toolbar-title>

        <div class="row q-gutter-x-md items-center">
          <QDarkMode />

          <div v-if="mxmConfig" class="column">
            <div class="row justify-between">
              <div class="row q-gutter-x-md items-center">
                <div>
                  Prototype: UI {{ mxmUiVersion }} / Server
                  {{ mxmConfig.mxmVersion }}
                </div>
              </div>
              <a
                v-if="mxmConfig.repoUrl"
                :href="mxmConfig.repoUrl"
                style="text-decoration: none"
                class="text-white q-ml-md"
                target="mxm_repo"
                rel="noopener"
              >
                <q-icon name="fab fa-github" />
              </a>
            </div>
            <div
              v-if="$q.screen.gt.sm"
              class="row q-gutter-x-md items-center"
              style="font-size: smaller"
            >
              <div class="row q-gutter-x-xs items-center">
                GraphQL:
                <div>
                  Endpoint
                  <q-tooltip>{{ mxmConfig.graphqlUri }}</q-tooltip>
                </div>
                <a
                  v-if="mxmConfig.graphqlSchema"
                  class="text-white"
                  style="text-decoration: none"
                  :href="mxmConfig.graphqlSchema"
                  target="mxm_graphqlSchema"
                  rel="noopener"
                >
                  Schema
                  <q-tooltip>{{ mxmConfig.graphqlSchema }}</q-tooltip>
                </a>
                <a
                  v-if="mxmConfig.graphqlUi"
                  style="font-size: small; text-decoration: none"
                  class="text-white"
                  :href="mxmConfig.graphqlUi"
                  target="mxm_graphqlUI"
                  rel="noopener"
                >
                  UI
                  <q-tooltip>{{ mxmConfig.graphqlUi }}</q-tooltip>
                </a>
              </div>
              <div
                v-if="mxmConfig.openapi"
                class="row q-gutter-x-xs items-center"
              >
                OpenAPI:
                <div>
                  endpoint
                  <q-tooltip>{{ mxmConfig.openapi }}</q-tooltip>
                </div>
                <a
                  v-if="mxmConfig.openapiSchema"
                  class="text-white"
                  style="text-decoration: none"
                  :href="mxmConfig.openapiSchema"
                  target="mxm_graphqlSchema"
                  rel="noopener"
                >
                  Schema
                  <q-tooltip>{{ mxmConfig.openapiSchema }}</q-tooltip>
                </a>
                <a
                  v-if="mxmConfig.swaggerUi"
                  style="font-size: small; text-decoration: none"
                  class="text-white"
                  :href="mxmConfig.swaggerUi"
                  target="mxm_swaggerUI"
                  rel="noopener"
                >
                  UI
                  <q-tooltip>{{ mxmConfig.swaggerUi }}</q-tooltip>
                </a>
              </div>
            </div>
          </div>
        </div>
      </q-toolbar>

      <UtlBreadcrumbs />
    </q-header>

    <q-page-container>
      <!--
      <pre class="q-ma-sm fieldBg">{{ mxmConfig }}</pre>
-->
      <router-view />
    </q-page-container>
  </q-layout>
</template>

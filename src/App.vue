<script setup lang="ts">
import { QLayout, QPageContainer, QPage } from 'quasar'
import { RouterView } from 'vue-router'
</script>

<template>
  <q-layout view="lHh Lpr lFf">
    <q-page-container>
      <q-page class="q-pa-md column items-start">
        <RouterView />
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<style scoped></style>

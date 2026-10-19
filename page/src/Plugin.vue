<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_sundtek_driver.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Container Selection -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-television-classic</v-icon>
          <span>{{ $t('plugin_sundtek_driver.select_container') }}</span>
        </v-card-title>
        <v-card-text>
          <v-select
            v-model="selectedContainer"
            :items="containers"
            :label="$t('plugin_sundtek_driver.select_container')"
            :loading="loadingContainers"
            :hint="$t('plugin_sundtek_driver.container_name_hint')"
            persistent-hint
            clearable
          >
            <template #item="{ item }">
              <div class="d-flex align-center ga-2">
                <v-icon size="18" :color="item.value ? 'success' : 'warning'">
                  {{ item.value.running ? 'mdi-play' : 'mdi-pause' }}
                </v-icon>
                <span>{{ item.title }}</span>
              </div>
            </template>
            <template #selection="{ item }">
              <div class="d-flex align-center ga-2">
                <v-icon size="18" :color="item.running ? 'success' : 'warning'">
                  {{ item.running ? 'mdi-play' : 'mdi-pause' }}
                </v-icon>
                <span>{{ item.name }}</span>
              </div>
            </template>
          </v-select>

          <div v-if="containers.length === 0" class="text-subtitle-1 text-error mt-4">
            <v-icon class="mr-2">mdi-alert-circle</v-icon>
            {{ $t('plugin_sundtek_driver.container_not_found') }}
          </div>
          <div v-if="containers.length === 0" class="text-caption text-medium-emphasis mt-1">
            {{ $t('plugin_sundtek_driver.container_not_found_hint') }}
          </div>
        </v-card-text>
      </v-card>

      <!-- Status Card -->
      <v-card v-if="selectedContainer" class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-information</v-icon>
          <span>{{ $t('plugin_sundtek_driver.container_status') }} - {{ selectedContainer.name }}</span>
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col cols="12" md="6">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_sundtek_driver.container_status') }}:
              </span>
              <v-chip :color="selectedContainer.running ? 'success' : 'warning'" class="ml-2">
                {{ selectedContainer.running ? $t('plugin_sundtek_driver.running') : $t('plugin_sundtek_driver.not_running') }}
              </v-chip>
            </v-col>
            <v-col cols="12" md="6">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_sundtek_driver.driver_status') }}:
              </span>
              <v-chip :color="selectedContainer.driver_installed ? 'success' : 'warning'" class="ml-2">
                {{ selectedContainer.driver_installed ? $t('plugin_sundtek_driver.driver_installed') : $t('plugin_sundtek_driver.driver_not_installed') }}
              </v-chip>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>

      <!-- Actions -->
      <v-card v-if="selectedContainer" class="mb-4 pa-0">
        <v-card-text class="d-flex align-center ga-2 flex-wrap">
          <v-btn
            color="primary"
            :loading="installing"
            :disabled="selectedContainer.driver_installed"
            @click="installDriver"
          >
            <v-icon start>mdi-download</v-icon>
            {{ $t('plugin_sundtek_driver.install_driver') }}
          </v-btn>
          <v-btn
            color="secondary"
            variant="tonal"
            :loading="updating"
            @click="updateDriver"
          >
            <v-icon start>mdi-refresh</v-icon>
            {{ $t('plugin_sundtek_driver.update_driver') }}
          </v-btn>
          <v-btn
            color="info"
            variant="tonal"
            :loading="updatingScript"
            @click="updateScript"
          >
            <v-icon start>mdi-download</v-icon>
            {{ $t('plugin_sundtek_driver.update_script') }}
          </v-btn>
        </v-card-text>
      </v-card>
    </div>

    <!-- Overlay -->
    <v-overlay :model-value="overlay" class="align-center justify-center">
      <div class="text-center">
        <v-progress-circular color="onPrimary" size="64" indeterminate />
        <div class="mt-4 text-subtitle-1">{{ overlayMessage }}</div>
      </div>
    </v-overlay>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const PLUGIN_NAME = 'sundtek-driver';

const loading = ref(true);
const loadingContainers = ref(false);
const overlay = ref(false);
const overlayMessage = ref('');
const installing = ref(false);
const updating = ref(false);
const updatingScript = ref(false);

const containers = ref([]);
const selectedContainer = ref(null);

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
  'Content-Type': 'application/json',
});

const queryPlugin = async (command, args = [], timeout = 30) => {
  const res = await fetch('/api/v1/mos/plugins/query', {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      command: PLUGIN_NAME,
      args: args,
      timeout: timeout,
      parse_json: false,
    }),
  });

  if (!res.ok) {
    throw new Error('Plugin query failed');
  }

  return await res.json();
};

// Query Docker API for all containers
const getDockerContainers = async () => {
  const res = await fetch('/api/v1/mos/docker/containers', {
    headers: getAuthHeaders(),
  });

  if (!res.ok) {
    throw new Error('Failed to get Docker containers');
  }

  return await res.json();
};

const refreshContainers = async () => {
  loadingContainers.value = true;
  try {
    const allContainers = await getDockerContainers();

    // Filter for TVHeadend containers (case-insensitive)
    const tvheadendContainers = allContainers.filter(c => {
      return c.name.toLowerCase().includes('tvheadend') || c.name.toLowerCase().includes('headend');
    });

    // Map to our format
    containers.value = tvheadendContainers.map(c => ({
      name: c.name,
      running: false, // Will be checked separately
      driver_installed: false,
    }));

    // Check status of each container
    for (let i = 0; i < containers.value.length; i++) {
      try {
        const result = await queryPlugin('status', [containers.value[i].name], 15);
        if (result.success && result.output) {
          containers.value[i].running = result.output.running;
          containers.value[i].driver_installed = result.output.driver_installed;
        }
      } catch (e) {
        console.error(`Failed to get status for ${containers.value[i].name}:`, e);
      }
    }

    // Try to restore previously selected container
    if (selectedContainer.value) {
      const found = containers.value.find(c => c.name === selectedContainer.value.name);
      if (found) {
        selectedContainer.value = found;
      }
    }
  } catch (e) {
    console.error('Failed to refresh containers:', e);
  } finally {
    loadingContainers.value = false;
  }
};

const installDriver = async () => {
  if (!selectedContainer.value) return;

  overlay.value = true;
  overlayMessage.value = `Installing driver to ${selectedContainer.value.name}...`;
  installing.value = true;

  try {
    await queryPlugin('install', [selectedContainer.value.name], 300);
    overlay.value = false;
    overlayMessage.value = '';
    await refreshContainers();
  } catch (e) {
    overlay.value = false;
    overlayMessage.value = '';
    console.error('Failed to install driver:', e);
  } finally {
    installing.value = false;
  }
};

const updateDriver = async () => {
  if (!selectedContainer.value) return;

  overlay.value = true;
  overlayMessage.value = `Updating driver in ${selectedContainer.value.name}...`;
  updating.value = true;

  try {
    await queryPlugin('update', [selectedContainer.value.name], 300);
    overlay.value = false;
    overlayMessage.value = '';
    await refreshContainers();
  } catch (e) {
    overlay.value = false;
    overlayMessage.value = '';
    console.error('Failed to update driver:', e);
  } finally {
    updating.value = false;
  }
};

const updateScript = async () => {
  overlay.value = true;
  overlayMessage.value = `Updating Sundtek script...`;
  updatingScript.value = true;

  try {
    await queryPlugin('update_script', [], 60);
    overlay.value = false;
    overlayMessage.value = '';
  } catch (e) {
    overlay.value = false;
    overlayMessage.value = '';
    console.error('Failed to update script:', e);
  } finally {
    updatingScript.value = false;
  }
};

onMounted(async () => {
  try {
    await refreshContainers();
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});
</script>

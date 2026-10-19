<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_sundtek_driver.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- No containers found -->
      <div v-if="containers.length === 0" class="text-center">
        <v-alert type="error" variant="tonal" class="mt-4">
          {{ $t('plugin_sundtek_driver.container_not_found') }}
        </v-alert>
        <div class="text-caption text-medium-emphasis mt-2">
          {{ $t('plugin_sundtek_driver.container_not_found_hint') }}
        </div>
        <div class="text-caption text-medium-emphasis">
          {{ $t('plugin_sundtek_driver.container_name_hint') }}
        </div>
      </div>

      <!-- Container cards -->
      <div v-else>
        <v-card
          v-for="container in containers"
          :key="container.name"
          class="mb-4 pa-0"
        >
          <v-card-title class="d-flex align-center">
            <v-icon
              size="24"
              :color="container.running ? 'success' : 'warning'"
              class="mr-2"
            >
              {{ container.running ? 'mdi-play' : 'mdi-pause' }}
            </v-icon>
            <span class="font-weight-medium">{{ container.name }}</span>
            <v-spacer />
            <v-chip
              :color="container.running ? 'success' : 'warning'"
              size="small"
              variant="tonal"
            >
              {{ container.running ? $t('plugin_sundtek_driver.running') : $t('plugin_sundtek_driver.not_running') }}
            </v-chip>
          </v-card-title>

          <v-divider />

          <v-card-actions class="d-flex align-center ga-2 pa-2">
            <v-btn
              color="primary"
              size="small"
              :loading="installing === container.name"
              :disabled="installing !== null"
              @click="startInstall(container)"
            >
              <v-icon start>mdi-download</v-icon>
              {{ $t('plugin_sundtek_driver.install_driver') }}
            </v-btn>
          </v-card-actions>
        </v-card>

        <!-- Update script button -->
        <v-card class="mb-4 pa-0">
          <v-card-text class="d-flex align-center ga-2">
            <v-btn
              color="info"
              size="small"
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
    </div>

    <!-- Installation Log Dialog -->
    <v-dialog v-model="showInstallLog" max-width="900" persistent>
      <v-card>
        <v-card-title class="d-flex align-center bg-grey-darken-3 text-white">
          <v-icon class="mr-2">mdi-console</v-icon>
          {{ $t('plugin_sundtek_driver.installation_log') }}: {{ installContainerName }}
          <v-spacer />
          <v-chip
            v-if="installStatus === 'running'"
            color="info"
            size="small"
            variant="tonal"
          >
            <v-icon start size="small">mdi-loading</v-icon>
            {{ $t('plugin_sundtek_driver.in_progress') }}
          </v-chip>
          <v-chip
            v-else-if="installStatus === 'success'"
            color="success"
            size="small"
            variant="tonal"
          >
            <v-icon start size="small">mdi-check</v-icon>
            {{ $t('plugin_sundtek_driver.completed') }}
          </v-chip>
          <v-chip
            v-else-if="installStatus === 'failed'"
            color="error"
            size="small"
            variant="tonal"
          >
            <v-icon start size="small">mdi-close</v-icon>
            {{ $t('plugin_sundtek_driver.failed') }}
          </v-chip>
        </v-card-title>

        <div
          ref="logContainer"
          style="
            background: #1a1a1a;
            color: #e0e0e0;
            font-family: 'Roboto Mono', monospace;
            font-size: 13px;
            padding: 16px;
            height: 450px;
            overflow-y: auto;
            white-space: pre-wrap;
            line-height: 1.5;
          "
        >
          {{ installLog }}
        </div>

        <v-card-actions>
          <v-spacer />
          <v-btn
            color="primary"
            :disabled="installStatus === 'running'"
            @click="closeInstallLog"
          >
            {{ $t('plugin_sundtek_driver.close') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import { io } from 'socket.io-client';

const PLUGIN_NAME = 'sundtek-driver';
const WS_PATH = '/api/v1/socket.io/';
const WS_URL = '/docker';

const loading = ref(true);
const installing = ref(null);
const updatingScript = ref(false);
const containers = ref([]);

const showInstallLog = ref(false);
const installLog = ref('');
const installStatus = ref('none');
const installPollTimer = ref(null);
const installContainerName = ref('');
const logContainer = ref(null);
let configureAttempted = ref(false);

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
      args: [command, ...args],
      timeout: timeout,
      parse_json: false,
    }),
  });

  if (!res.ok) {
    throw new Error('Plugin query failed');
  }

  return await res.json();
};

const refreshContainers = async () => {
  loading.value = true;
  try {
    const res = await fetch('/api/v1/docker/containers/json?all=true', {
      headers: getAuthHeaders(),
    });
    if (!res.ok) {
      throw new Error('Failed to get Docker containers');
    }

    const allContainers = await res.json();

    containers.value = allContainers
      .filter(c => {
        const name = c.Names[0].replace(/^\//, '');
        return name.toLowerCase().includes('tvheadend') || name.toLowerCase().includes('headend');
      })
      .map(c => {
        const name = c.Names[0].replace(/^\//, '');
        return {
          name: name,
          running: c.State === 'running',
        };
      });
  } catch (e) {
    console.error('Failed to refresh containers:', e);
  } finally {
    loading.value = false;
  }
};

const appendLog = (message) => {
  installLog.value += message + '\n';
  nextTick(() => {
    if (logContainer.value) {
      logContainer.value.scrollTop = logContainer.value.scrollHeight;
    }
  });
};

// Configure container via Socket.IO
const configureContainer = async (containerName) => {
  appendLog('');
  appendLog('=== Configuring Container ===');

  try {
    appendLog('Fetching container template...');
    const templateRes = await fetch(
      `/api/v1/docker/mos/templates/${containerName}?edit=true`,
      { headers: getAuthHeaders() }
    );
    if (!templateRes.ok) {
      throw new Error(`Failed to get template (HTTP ${templateRes.status})`);
    }
    let template = await templateRes.json();
    appendLog('Template loaded');

    // Add USB device passthrough if not present
    const hasUsb = template.devices.some(d => d.host === '/dev/bus/usb');
    if (!hasUsb) {
      template.devices.push({
        name: 'usb',
        host: '/dev/bus/usb',
        container: '/dev/bus/usb',
        description: 'USB passthrough for Sundtek tuner',
      });
      appendLog('Added /dev/bus/usb passthrough');
    }

    // Add DVB device passthrough if not present
    const hasDvb = template.devices.some(d => d.host === '/dev/dvb');
    if (!hasDvb) {
      template.devices.push({
        name: 'dvb',
        host: '/dev/dvb',
        container: '/dev/dvb',
        description: 'DVB passthrough for Sundtek tuner',
      });
      appendLog('Added /dev/dvb passthrough');
    }

    // Add wrapper script mount if not present
    const hasWrapper = template.paths.some(
      p => p.container === '/etc/cont-init.d/99-sundtekdriverinstall'
    );
    if (!hasWrapper) {
      template.paths.push({
        name: 'sundtek-driver-wrapper',
        host: '/var/mos/plugins/sundtek-plugins/99-sundtekdriverinstall',
        container: '/etc/cont-init.d/99-sundtekdriverinstall',
        mode: '',
        description: 'Auto-install Sundtek driver on container start',
      });
      appendLog('Added wrapper script mount');
    }

    // Send update via Socket.IO
    appendLog('Updating container via Docker API...');
    await sendDockerWSCommand('create', { template: template });
    appendLog('Container configuration completed!');
    appendLog('');
    appendLog('=== All done ===');

    // Refresh container list after update
    await new Promise(resolve => setTimeout(resolve, 3000));
    await refreshContainers();
  } catch (e) {
    appendLog(`ERROR: ${e.message}`);
  }
};

const sendDockerWSCommand = (command, params) => {
  return new Promise((resolve, reject) => {
    const authToken = localStorage.getItem('authToken');
    const socket = io(WS_URL, { path: WS_PATH, transports: ['websocket'] });

    let resolved = false;

    socket.on('connect', () => {
      socket.emit('docker', {
        token: authToken,
        operation: command,
        params: params,
      });
    });

    socket.on('docker-update', (data) => {
      if (data.output) {
        appendLog(data.output);
      }
      if (data.status === 'completed') {
        resolved = true;
        socket.disconnect();
        resolve(data);
      } else if (data.status === 'error') {
        resolved = true;
        socket.disconnect();
        reject(new Error(data.output || 'Docker operation failed'));
      }
    });

    socket.on('connect_error', (err) => {
      if (!resolved) {
        resolved = true;
        socket.disconnect();
        reject(new Error('WebSocket connection failed: ' + err.message));
      }
    });

    // Timeout after 120 seconds
    setTimeout(() => {
      if (!resolved && socket.connected) {
        resolved = true;
        socket.disconnect();
        reject(new Error('Docker operation timed out'));
      }
    }, 120000);
  });
};

const startInstallLogPolling = (containerName) => {
  installContainerName.value = containerName;
  installLog.value = '';
  installStatus.value = 'running';
  showInstallLog.value = true;
  configureAttempted.value = false;

  const poll = async () => {
    try {
      const statusResult = await queryPlugin('install_status', [containerName], 10);
      if (statusResult.success && statusResult.output) {
        const status = JSON.parse(statusResult.output);
        installStatus.value = status.status;
      }

      const logResult = await queryPlugin('install_log', [containerName], 10);
      if (logResult.success && logResult.output) {
        installLog.value = logResult.output;

        await nextTick();
        if (logContainer.value) {
          logContainer.value.scrollTop = logContainer.value.scrollHeight;
        }
      }

      if (installStatus.value === 'running') {
        installPollTimer.value = setTimeout(poll, 1000);
      } else if (installStatus.value === 'success' && !configureAttempted.value) {
        // Backend installation done - now configure container via Socket.IO
        configureAttempted.value = true;
        installStatus.value = 'running'; // Keep dialog open
        await configureContainer(containerName);
        installStatus.value = 'success';
      }
    } catch (e) {
      console.error('Failed to poll install status:', e);
    }
  };

  poll();
};

const startInstall = async (container) => {
  installing.value = container.name;
  try {
    await queryPlugin('install', [container.name], 10);
    startInstallLogPolling(container.name);
  } catch (e) {
    console.error('Failed to start installation:', e);
    installing.value = null;
  }
};

const closeInstallLog = async () => {
  if (installPollTimer.value) {
    clearTimeout(installPollTimer.value);
    installPollTimer.value = null;
  }
  showInstallLog.value = false;
  installing.value = null;
  installStatus.value = 'none';
  installLog.value = '';
  await refreshContainers();
};

const updateScript = async () => {
  updatingScript.value = true;
  try {
    await queryPlugin('update_script', [], 60);
  } catch (e) {
    console.error('Failed to update script:', e);
  } finally {
    updatingScript.value = false;
  }
};

onMounted(async () => {
  await refreshContainers();
  loading.value = false;
});

onUnmounted(() => {
  if (installPollTimer.value) {
    clearTimeout(installPollTimer.value);
  }
});
</script>

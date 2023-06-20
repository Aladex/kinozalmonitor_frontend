<template>
  <v-card class="mx-auto" max-width="1800">
    <v-card-title><v-icon>mdi-download</v-icon>
      Torrents</v-card-title>
    <v-card-text>
      <v-banner
        v-for="torrent in torrents"
        :key="torrent.id"
      >
        <v-banner-text>
          <v-row>
            <v-col cols="12" sm="12" md="4" lg="4">
              <div class="align-left">
                <a :href="torrent.url">{{ torrent.title }}</a>
              </div>
            </v-col>
            <v-col cols="12" sm="12" md="4" lg="4">
              <div class="align-center">
                <p>Hash: {{ torrent.hash }}</p>
              </div>
            </v-col>
            <v-col cols="12" sm="12" md="1" lg="1">
                <v-select
                  class="align-center"
                  v-model="torrent.watch_every"
                  :items="watchEveryOptions"
                  label="Watch every"
                  item-value="item"
                  variant="underlined"
                  @update:modelValue="updateWatchEvery(torrent)"
                ></v-select>
            </v-col>
            <v-col cols="12" sm="12" md="3" lg="3">
              <div class="align-right">
                <v-btn color="primary" @click="deleteTorrent(torrent)">
                  Delete
                </v-btn>
              </div>
            </v-col>
          </v-row>
        </v-banner-text>
      </v-banner>
    </v-card-text>
    <v-card-actions>
      <v-btn
        color="primary"
      >
        Add torrent

        <v-dialog
          v-model="dialogVisible"
          activator="parent"
          width="500"
        >
          <v-card>
            <v-card-title>Add Torrent</v-card-title>
            <v-card-text>
              <v-form @submit.prevent="addTorrent">
                <v-text-field v-model="torrentUrl" label="Torrent URL"></v-text-field>
                <v-select
                  v-model="selectedDownloadPath"
                  :items="downloadPaths"
                  label="Download path"
                  item-text="path"
                  item-value="path"
                  :required="isDownloadPathRequired"
                ></v-select>
                <v-btn type="submit" color="primary" :disabled="loading">
                  <v-progress-circular v-if="loading" indeterminate size="14" class="mr-2"></v-progress-circular>
                  Add
                </v-btn>
              </v-form>
            </v-card-text>
          </v-card>
        </v-dialog>
      </v-btn>
    </v-card-actions>
  </v-card>
  <v-overlay
    v-model="loading"
    absolute="true"
    class="align-center justify-center"
  >
    <v-progress-circular indeterminate size="64"></v-progress-circular>
  </v-overlay>
  <v-dialog v-model="warningDialogVisible" max-width="400">
    <v-card>
      <v-card-title class="headline">Warning</v-card-title>
      <v-card-text>
        Мы попали под ограничение количества скачанных торрентов. Торрент будет добавлен позже автоматически.
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="green darken-1" text @click="warningDialogVisible = false">
          OK
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
  <v-dialog v-model="errorDialogVisible" max-width="400">
    <v-card>
      <v-card-title class="headline">Error</v-card-title>
      <v-card-text>
        Что-то пошло не так. Возможно, линк неверный.
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="green darken-1" text @click="errorDialogVisible = false">
          OK
        </v-btn>
      </v-card-actions>

    </v-card>
  </v-dialog>

</template>

<script>
export default {
  name: "TorrentList",
  data() {
    return {
      torrents: [],
      dialogVisible: false,
      torrentUrl: "",
      warningDialogVisible: false,
      errorDialogVisible: false,
      apiUrl: import.meta.env.VITE_BACKEND_API,
      loading: false,
      downloadPaths: [],
      selectedDownloadPath: null,
    };
  },
  computed: {
    isDownloadPathRequired() {
      return this.downloadPaths.length === 0;
    },
    // In minutes
    watchEveryOptions() {
      // Create list of options from 10 to 240 minutes with step 30 minutes
      const options = ["-"];
      for (let i = 10; i <= 240; i += 30) {
        options.push(i.toString());
      }
      return options;
    }
  },
  methods: {
    async updateWatchEvery(torrent) {
      // Convert "-" back to 0 for API
      const watchPeriod = torrent.watch_every === "-" ? "0" : torrent.watch_every;
      try {
        const response = await fetch(this.apiUrl + "api/watch", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            url: torrent.url,
            watchPeriod: watchPeriod,
          }),
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        if (data.status !== 'ok') {
          console.error("Error setting watch flag: ", data.error);
        }
      } catch (error) {
        console.error("Error: ", error);
      }
    },
    async getTorrents() {
      const response = await fetch(this.apiUrl + "api/torrents");
      this.torrents = await response.json();
      // Iterate over torrents and set the watchEvery value to "-" if it's === 0
      this.torrents.forEach((torrent) => {
        if (torrent.watch_every === 0) {
          torrent.watch_every = "-";
        }
      });
    },
    async getDownloadPaths() {
      const response = await fetch(this.apiUrl + "api/download-paths");
      this.downloadPaths = await response.json();
      // Set the first path as selected
      if (this.downloadPaths.length > 0) {
        this.selectedDownloadPath = this.downloadPaths[0];
      }
    },
    async deleteTorrent(item) {
      try {
        const response = await fetch(this.apiUrl + "api/remove", {
          method: "DELETE",
          headers: {
            "Content-Type": "application/json"
          },
          body: JSON.stringify({
            url: item.url,
            hash: item.hash,
          })
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();

        if (data.status === "ok") {
          // Remove the item from the local list
          this.torrents = this.torrents.filter(torrent => torrent.name !== item.name);
        } else {
          console.error("Error removing torrent: ", data.error);
        }
      } catch (error) {
        console.error("Error: ", error);
      }
      // Update the list of torrents
      this.getTorrents();
    },
    async addTorrent() {
      if (this.isDownloadPathRequired && this.selectedDownloadPath === "") {
        this.errorDialogVisible = true;
        throw new Error("Download path is required");
      }
      try {
        this.loading = true;
        const payload = {
          url: this.torrentUrl,
          downloadPath: this.selectedDownloadPath
        };

        const response = await fetch(this.apiUrl + "api/add", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(payload),
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        if (data.status !== 'ok') {
          this.errorDialogVisible = true;
          console.error("Error adding torrent: ", data.error);
        }
      } catch (error) {
        console.error("Error: ", error);
        this.errorDialogVisible = true;
      } finally {
        this.loading = false;
        this.dialogVisible = false;
        this.torrentUrl = "";
        this.getTorrents();
      }
    },
  },
  mounted() {
    this.getTorrents();

    // Create WebSocket url from the current url if it's not set in .env
    this.wsUrl = import.meta.env.VITE_BACKEND_WS || `ws://${window.location.host}/ws`;

    // if https then use wss
    if (window.location.protocol === "https:") {
      this.wsUrl = this.wsUrl.replace("ws:", "wss:");
    }

    this.socket = new WebSocket(this.wsUrl);

    // Connection opened
    this.socket.onmessage = (event) => {
      const msg = event.data;

      // if something was added then update the list of torrents
      if (msg === "added") {
        this.loading = false;
        this.getTorrents();
      } else if (msg === "500") {
        this.loading = false;
        this.errorDialogVisible = true;
      }
    };
  },
  watch: {
    dialogVisible(val) {
      if (!val) {
        this.torrentUrl = "";
      } else {
        this.getDownloadPaths();
      }
    },
  },
};
</script>

<style scoped>
.align-left {
  text-align: left;
}
.align-center {
  text-align: center;
}
.align-right {
  text-align: right;
}
</style>

<template>
  <v-card class="container">
    <v-container fluid>
      <v-row>
        <v-col cols="12">
          <v-breadcrumbs :items="breadcrumbs">
            <template v-slot:divider>
              <v-icon>mdi-chevron-right</v-icon>
            </template>
          </v-breadcrumbs>
        </v-col>
      </v-row>
      <v-row v-if="hasImages">
        <v-col cols="12">
          <v-carousel
            height="320"
            hide-delimiter-background
            hide-delimiters
            show-arrows-on-hover>

            <v-carousel-item
              v-for="image in images">
              <v-list-item :href="'https://link12.ddns.net:9090' + image.fileUrl" target="_blank">
                <v-list-item-title>
                  <v-img :src="'https://link12.ddns.net:9090' + image.thumbUrl"
                    max-width="320" max-height="240" :style="{'margin': '20px auto'}"></v-img>
                </v-list-item-title>
              </v-list-item>
            </v-carousel-item>
          </v-carousel>
        </v-col>
      </v-row>
      <v-row>
        <v-col cols="12">
          <FilesList>
            <template #item="{ id, itemId, dataUrl }">
              <v-list>
                <v-list-item link @click="handleClick($event, itemId)">
                  <v-list-item-title>
                    <v-icon>mdi-console-line</v-icon>
                    {{ dataUrl }}
                  </v-list-item-title>

                  <v-list-item-subtitle class="video" v-bind:id="'player-' + itemId">
                    <v-progress-linear
                      indeterminate
                      reverse
                      stream
                      v-show="loading"
                      color="blue">
                    </v-progress-linear>
                  </v-list-item-subtitle>
                </v-list-item>
                <v-divider inset></v-divider>
              </v-list>
            </template>
          </FilesList>
        </v-col>
      </v-row>
    </v-container>
  </v-card>
</template>

<script>
  import { mapMutations } from 'vuex';
  import showItem from '../mutations/showItem';
  import FilesList from '../components/FilesList.vue';
  import _ from 'lodash';
  import _get from 'lodash/get';
  import axios from 'axios';

  export default {
    name: 'LoadFolder',
    components: { FilesList },
    created() {
      this.loadFolder();
    },
    data () {
      return {
        intervals: {},
        loading: false,
        breadcrumbs: [],
      };
    },
    methods: {
      ...mapMutations(['setStoreData']),
      loadFolder() {
        showItem({
          apollo: this.$apollo,
          id: this.$route.params.id,
        }).then((response) => _get(response, 'data.showItem', {}))
          .then((response) => {
            this.setStoreData({
              'displayItem': {
                folder: response.folder,
                audioFiles: response.audioFiles,
                images: response.images,
              }
            });
            this.showBreadCrumbs();
          }).catch((error) => {
            this.$toast.warning(this.$t('error.unknonwn'));
          });
      },
      showBreadCrumbs() {
        this.breadcrumbs = [];
        this.breadcrumbs.push({
          disabled: false,
          text: this.$t('home.title'),
          to: {
            path: `/home`,
          }
        });
        if (this.storeData.search.q) {
          this.breadcrumbs.push({
            disabled: false,
            text: this.storeData.search.q,
            to: { path: `/home` },
          });
        } else if (this.currentFolder.parentFolder && this.currentFolder.parentSubfolder) {
          if (this.currentFolder.parentFolder != 'N/A') {
            this.breadcrumbs.push({
              disabled: false,
              text: this.currentFolder.parentFolder,
              to: {
                path: `/home/${this.currentFolder.parentFolder}`,
              }
            });
          }
          if (this.currentFolder.parentSubfolder != 'N/A') {
            this.breadcrumbs.push({
              disabled: false,
              text: this.currentFolder.parentSubfolder,
              to: {
                path: `/home/${this.currentFolder.parentFolder}/${this.currentFolder.parentSubfolder}`,
              }
            });
          }
        } else if (this.currentFolder.parentFolder) {
          if (this.currentFolder.parentFolder != 'N/A') {
            this.breadcrumbs.push({
              disabled: false,
              text: this.currentFolder.parentFolder,
              to: {
                path: `/home/${this.currentFolder.parentFolder}`,
              }
            });
          }
        }
        this.breadcrumbs.push({
          disabled: false,
          text: this.currentFolder.name,
          to: {
            path: `/${this.currentFolder.dataUrl}`,
          },
        });
      },
      railsImportExternalUrl(itemId) {
        return 'https://link12.ddns.net:3000/audio_files/' + itemId + '.json';
      },
      hlsServerExternalUrl(itemId) {
        return 'https://link12.ddns.net:3000/hls/' + itemId + '.m3u8';
      },
      buildIframe(url) {
        let iframe = document.createElement('iframe');
        iframe.src = url;
        return iframe;
      },
      clearIframes() {
        _.map(document.getElementsByClassName('video'), function(el) {
          _.map(el.getElementsByTagName('iframe'), function(iframe) {
            iframe.src = '';
          });
        });
      },
      showPlayer(itemId) {
        _.map(document.getElementsByClassName('video'), function(el) {
          el.style.display = el.id == 'player-' + itemId ? 'block' : 'none';
        });
      },
      getPlayer(itemId) {
        return document.getElementById('player-' + itemId);
      },
      handleClick(event, itemId) {
        if (this.loading) return;
        if (this.getPlayer(itemId).getElementsByTagName('iframe').length > 0) return;
        this.loading = true;
        this.clearIframes();
        this.showPlayer(itemId);
        this.intervals[itemId] = setInterval(() => {
          axios.get(this.railsImportExternalUrl(itemId))
          .then((response) => _get(response, 'data', {}))
          .then((response) => {
            if (response.m3u8_exists) {
              clearInterval(this.intervals[itemId]);
              this.getPlayer(itemId).appendChild(
                this.buildIframe(
                  this.hlsServerExternalUrl(itemId)
                )
              );
              this.loading = false;
            }
          })
          .catch((error) => {
            this.$toast.warning(this.$t('error.unknonwn'));
          })
        }, 2000);
      },
    },
  };
</script>

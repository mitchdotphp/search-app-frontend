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
      <v-row>
        <v-col cols="12">
          <v-text-field
            v-bind:label="$t('search.input_label')"
            class="pa-1 ma-1"
            color="blue darken-2"
            outlined
            variant="outlined"
            clearable
            clear-icon="mdi-delete"
            @click:clear="handleClickClear"
            @keydown.enter="handleSearchFromInput"
            v-model="searchQuery">
          </v-text-field>
        </v-col>
      </v-row>
      <v-row v-if="hasResults">
        <v-col cols="12">
          <SearchResults>
            <template #item="{ id, dataUrl, name, parentFolder, parentSubfolder, year }">
              <v-list>
                <v-list-item>
                  <!-- Folder / Subfolder -->
                  <v-list-item-title>
                    <v-icon>mdi-folder-open</v-icon>

                    <v-chip
                      color="grey darken-2"
                      @click="handleClickYear($event, dataUrl, year)"
                      class="ma-1"
                      x-small
                      outlined
                      label>
                      {{ year }}
                    </v-chip>

                    <v-chip
                      color="blue darken-2"
                      @click="handleClickFolder($event, dataUrl, parentFolder)"
                      v-if="parentFolder != 'N/A'"
                      class="ma-1"
                      x-small
                      outlined
                      label>
                      {{ parentFolder }}
                    </v-chip>

                    <v-chip
                      color="purple darken-2"
                      @click="handleClickSubfolder($event, dataUrl, parentFolder, parentSubfolder)"
                      v-if="parentSubfolder != 'N/A'"
                      class="ma-1"
                      x-small
                      outlined
                      label>
                      {{ parentSubfolder }}
                    </v-chip>
                  </v-list-item-title>

                  <!-- Name -->
                  <v-list-item-title
                    @click="handleClick($event, dataUrl)"
                    class="ma-2 cursorPointer">
                    {{ name }}
                  </v-list-item-title>
                </v-list-item>
              </v-list>
            </template>
          </SearchResults>
        </v-col>
      </v-row>
      <v-row v-if="!hasResults" align="center" justify="center" class="noResults">
        <v-col cols="12">
          {{ $t('search.no_results') }}
        </v-col>
      </v-row>
      <v-row v-if="hasResults" align="center" justify="center">
        <v-col cols=12 class="ma-5 pa-5">
          <v-pagination
            color="blue darken-2"
            :size="10"
            :show-first-last-page="true"
            :total-visible="10"
            v-model:model-value="currentPage"
            v-model:length="totalPages"
            @update:modelValue="handleSearchFromPagination">
          </v-pagination>
        </v-col>
      </v-row>
    </v-container>
  </v-card>
</template>

<script>
  import { mapMutations } from 'vuex';
  import SearchResults from '../components/SearchResults.vue';
  import search from '../mutations/search';
  import _ from 'lodash';
  import _get from 'lodash/get';

  export default {
    name: 'SearchTable',
    components: { SearchResults },
    created() {
      // created
    },
    data() {
      return {
        breadcrumbs: [],
        defaultSearchParams: {
          page: 1,
          q: '',
          folder: '',
          subfolder: '',
        },
        search: {},
      };
    },
    watch:{
      '$route.name': {
        handler: function(route_name) {
          switch (route_name) {
            // Home
            case 'home': {
              _.assign(this.search, this.storeData.search);
              this.handleSearch();
              this.breadcrumbs = [];
              this.breadcrumbs.push({
                disabled: false,
                text: this.$t('home.title'),
                to: {
                  name: 'home_redirect',
                  state: this.defaultSearchParams,
                },
              });
              if (this.storeData && this.storeData.search && this.storeData.search.q) {
                this.breadcrumbs.push({
                  disabled: false,
                  text: this.storeData.search.q,
                  to: { path: `/home` },
                });
              }
              break;
            }

            // Home
            case 'home_redirect': {
              _.assign(this.search, this.defaultSearchParams);
              this.handleSearch();
              this.breadcrumbs = [];
              this.breadcrumbs.push({
                disabled: false,
                text: this.$t('home.title'),
                to: {
                  name: 'home_redirect',
                  state: this.defaultSearchParams,
                },
              });
              break;
            }

            // Search
            case 'home_search': {
              const query = this.$route.params.q;
              _.assign(this.search, this.defaultSearchParams, { q: query });
              this.handleSearch();
              this.breadcrumbs = [];
              this.breadcrumbs.push({
                disabled: false,
                text: this.$t('home.title'),
                to: {
                  name: 'home_redirect',
                  state: this.defaultSearchParams,
                },
              });
              if (this.storeData && this.storeData.search && this.storeData.search.q) {
                this.breadcrumbs.push({
                  disabled: false,
                  text: this.storeData.search.q,
                  to: { path: `/home` },
                });
              }
              break;
            }

            // Browse Subfolder
            case 'home_folder_subfolder': {
              const folder = this.$route.params.folder;
              const subfolder = this.$route.params.subfolder;
              _.assign(this.search, this.defaultSearchParams, { folder: folder, subfolder: subfolder });
              this.handleSearch();
              this.breadcrumbs = [{
                disabled: false,
                text: this.$t('home.title'),
                to: {
                  name: 'home_redirect',
                  state: this.defaultSearchParams,
                },
              },
              {
                disabled: false,
                text: folder,
                to: {
                  path: `/home/${folder}`,
                },
              },
              {
                disabled: false,
                text: subfolder,
                to: {
                  path: `/home/${folder}/${subfolder}`,
                },
              }];
              break;
            }

            // Browse Folder
            case 'home_folder': {
              const folder = this.$route.params.folder;
              _.assign(this.search, this.defaultSearchParams, { folder: folder });
              this.handleSearch();
              this.breadcrumbs = [{
                disabled: false,
                text: this.$t('home.title'),
                to: {
                  name: 'home_redirect',
                  state: this.defaultSearchParams,
                },
              },
              {
                disabled: false,
                text: folder,
                to: {
                  path: `/home/${folder}`,
                },
              }];
              break;
            }

            case 'show': {
              // Component is replaced by DisplayItem
              break;
            }

            default: {
            // Route not found
              this.$toast.warning('Route not found: ' + route_name);
              break;
            }
          }
        },
        deep: true,
        immediate: true,
      },
    },
    methods: {
      ...mapMutations(['setStoreData']),
      handleSearch() {
      search(_.assign({ apollo: this.$apollo }, this.defaultSearchParams, this.search))
        .then((response) => _get(response, 'data.search', {}))
        .then((response) => {
          if (response) {
            this.setStoreData({
              'search': _.assign(this.search, {
                response: response,
              })
            });
          }
        }).catch((error) => {
          this.$toast.warning('Unknown error');
        });
      },
      handleClick(event, dataUrl) {
        this.$router.push({
          path: `/${dataUrl}`,
        });
      },
      handleClickClear(event) {
        this.$router.push({
          name: `home_redirect`,
          state: this.defaultSearchParams,
        });
      },
      handleSearchFromInput(event) {
        this.$router.push({
          path: `/search/${event.target.value}`,
        });
      },
      handleSearchFromPagination(event) {
        _.assign(this.search, this.storeData.search, { page: event });
        this.handleSearch();
      },
      handleClickFolder(event, dataUrl, folder) {
        this.$router.push({
          path: `/home/${folder}`,
        });
      },
      handleClickSubfolder(event, dataUrl, folder, subfolder) {
        this.$router.push({
          path: `/home/${folder}/${subfolder}`,
        });
      },
      handleClickYear(event, dataUrl, year) {
        this.$router.push({
          path: `/search/${year}`,
        });
      }
    },
  };
</script>

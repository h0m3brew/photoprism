<template>
    <div class="p-page p-page-album-photos" v-infinite-scroll="loadMore" :infinite-scroll-disabled="scrollDisabled"
         :infinite-scroll-distance="10" :infinite-scroll-listen-for-event="'scrollRefresh'">

        <p-album-toolbar :album="model" :settings="settings" :filter="filter" :filter-change="updateQuery"
                         :refresh="refresh"></p-album-toolbar>

        <v-container fluid class="pa-4" v-if="loading">
            <v-progress-linear color="secondary-dark" :indeterminate="true"></v-progress-linear>
        </v-container>
        <v-container fluid class="pa-0" v-else>
            <p-scroll-top></p-scroll-top>

            <p-photo-clipboard :refresh="refresh"
                               :selection="selection"
                               :album="model" context="album"></p-photo-clipboard>

            <p-photo-mosaic v-if="settings.view === 'mosaic'"
                            :photos="results"
                            :selection="selection"
                            :album="model"
                            :edit-photo="editPhoto"
                            :open-photo="openPhoto"></p-photo-mosaic>
            <p-photo-list v-else-if="settings.view === 'list'"
                          :photos="results"
                          :selection="selection"
                          :album="model"
                          :open-photo="openPhoto"
                          :edit-photo="editPhoto"
                          :open-location="openLocation"></p-photo-list>
            <p-photo-cards v-else
                           :photos="results"
                           :selection="selection"
                           :album="model"
                           :open-photo="openPhoto"
                           :edit-photo="editPhoto"
                           :open-location="openLocation"></p-photo-cards>
        </v-container>
    </div>
</template>

<script>
    import Photo from "model/photo";
    import Album from "model/album";
    import Event from "pubsub-js";
    import Thumb from "model/thumb";

    export default {
        name: 'p-page-album-photos',
        props: {
            staticFilter: Object
        },
        watch: {
            '$route'() {
                const query = this.$route.query;

                this.filter.q = query['q'] ? query['q'] : '';
                this.filter.camera = query['camera'] ? parseInt(query['camera']) : 0;
                this.filter.country = query['country'] ? query['country'] : '';
                this.settings.view = this.viewType();
                this.lastFilter = {};
                this.routeName = this.$route.name;

                if (this.uuid !== this.$route.params.uuid) {
                    this.uuid = this.$route.params.uuid;
                    this.findAlbum().then(() => this.search());
                } else {
                    this.search();
                }
            }
        },
        data() {
            const uuid = this.$route.params.uuid;
            const query = this.$route.query;
            const routeName = this.$route.name;
            const order = query['order'] ? query['order'] : 'oldest';
            const camera = query['camera'] ? parseInt(query['camera']) : 0;
            const q = query['q'] ? query['q'] : '';
            const country = query['country'] ? query['country'] : '';
            const view = this.viewType();
            const filter = {country: country, camera: camera, order: order, q: q};
            const settings = {view: view};

            return {
                subscriptions: [],
                listen: false,
                dirty: false,
                model: new Album(),
                uuid: uuid,
                results: [],
                scrollDisabled: true,
                pageSize: 60,
                offset: 0,
                page: 0,
                selection: this.$clipboard.selection,
                settings: settings,
                filter: filter,
                lastFilter: {},
                routeName: routeName,
                loading: true,
            };
        },
        methods: {
            viewType() {
                let queryParam = this.$route.query['view'];
                let defaultType = window.localStorage.getItem("photo_view_type");
                let storedType = window.localStorage.getItem("album_view_type");

                if (queryParam) {
                    window.localStorage.setItem("album_view_type", queryParam);
                    return queryParam;
                } else if (storedType) {
                    return storedType;
                } else if (defaultType) {
                    return defaultType;
                } else if (window.innerWidth < 960) {
                    return 'mosaic';
                }

                return 'cards';
            },
            openLocation(index) {
                const photo = this.results[index];

                if (photo.LocationID) {
                    this.$router.push({name: "place", params: {q: "s2:" + photo.LocationID}});
                } else if (photo.PlaceID.length > 3) {
                    this.$router.push({name: "place", params: {q: "s2:" + photo.PlaceID}});
                }
            },
            editPhoto(index) {
                let selection = this.results.map((p) => {
                    return p.getId()
                });

                // Open Edit Dialog
                Event.publish("dialog.edit", {selection: selection, album: this.album, index: index});
            },
            openPhoto(index, showMerged) {
                if(!this.results[index]) {
                    return false;
                }

                if (showMerged && this.results[index].PhotoVideo) {
                    if(this.results[index].isPlayable()) {
                        Event.publish("dialog.video", {play: this.results[index], album: this.album});
                    } else {
                        this.$viewer.show(Thumb.fromPhotos(this.results), index);
                    }
                } else if (showMerged) {
                    this.$viewer.show(Thumb.fromFiles([this.results[index]]), 0)
                } else {
                    this.$viewer.show(Thumb.fromPhotos(this.results), index);
                }

                return true;
            },
            loadMore() {
                if (this.scrollDisabled) return;

                this.scrollDisabled = true;
                this.listen = false;

                const count = this.dirty ? (this.page + 2) * this.pageSize : this.pageSize;
                const offset = this.dirty ? 0 : this.offset;

                const params = {
                    count: count,
                    offset: offset,
                    album: this.uuid,
                    merged: true,
                };

                Object.assign(params, this.lastFilter);

                if (this.staticFilter) {
                    Object.assign(params, this.staticFilter);
                }

                Photo.search(params).then(response => {
                    this.results = Photo.mergeResponse(this.results, response);

                    this.scrollDisabled = (response.models.length < count);

                    if (this.scrollDisabled) {
                        this.offset = offset;

                        if (this.results.length > 1) {
                            this.$notify.info(this.$gettext('All ') + this.results.length + this.$gettext(' photos loaded'));
                        }
                    } else {
                        this.offset = offset + count;
                        this.page++;
                    }
                }).catch(() => {
                    this.scrollDisabled = false;
                }).finally(() => {
                    this.dirty = false;
                    this.loading = false;
                    this.listen = true;
                });
            },
            updateQuery() {
                const query = {
                    view: this.settings.view
                };

                Object.assign(query, this.filter);

                for (let key in query) {
                    if (query[key] === undefined || !query[key]) {
                        delete query[key];
                    }
                }

                if (JSON.stringify(this.$route.query) === JSON.stringify(query)) {
                    return
                }

                this.$router.replace({query: query});
            },
            searchParams() {
                const params = {
                    count: this.pageSize,
                    offset: this.offset,
                    album: this.uuid,
                    merged: true,
                };

                Object.assign(params, this.filter);

                if (this.staticFilter) {
                    Object.assign(params, this.staticFilter);
                }

                return params;
            },
            refresh() {
                if (this.loading) return;
                this.loading = true;
                this.page = 0;
                this.dirty = true;
                this.scrollDisabled = false;
                this.loadMore();
            },
            search() {
                this.scrollDisabled = true;

                // Don't query the same data more than once
                if (JSON.stringify(this.lastFilter) === JSON.stringify(this.filter)) {
                    this.$nextTick(() => this.$emit("scrollRefresh"));
                    return;
                }

                Object.assign(this.lastFilter, this.filter);

                this.offset = 0;
                this.page = 0;
                this.loading = true;
                this.listen = false;

                const params = this.searchParams();

                Photo.search(params).then(response => {
                    this.offset = this.pageSize;

                    this.results = response.models;

                    this.scrollDisabled = (response.models.length < this.pageSize);

                    if (this.scrollDisabled) {
                        if (!this.results.length) {
                            this.$notify.warning(this.$gettext("No photos found"));
                        } else if (this.results.length === 1) {
                            this.$notify.info(this.$gettext("One photo found"));
                        } else {
                            this.$notify.info(this.results.length + this.$gettext(" photos found"));
                        }
                    } else {
                        this.$notify.info(this.$gettext('More than 50 photos found'));

                        this.$nextTick(() => this.$emit("scrollRefresh"));
                    }
                }).finally(() => {
                    this.dirty = false;
                    this.loading = false;
                    this.listen = true;
                });
            },
            findAlbum() {
                return this.model.find(this.uuid).then(m => {
                    this.model = m;

                    this.filter.order = m.AlbumOrder;
                    window.document.title = `PhotoPrism: ${this.model.AlbumName}`;

                    return Promise.resolve(this.model)
                });
            },
            onAlbumsUpdated(ev, data) {
                if (!this.listen) return;

                if (!data || !data.entities) {
                    return
                }

                for (let i = 0; i < data.entities.length; i++) {
                    if (this.model.AlbumUUID === data.entities[i].AlbumUUID) {
                        let values = data.entities[i];

                        for (let key in values) {
                            if (values.hasOwnProperty(key)) {
                                this.model[key] = values[key];
                            }
                        }

                        window.document.title = `PhotoPrism: ${this.model.AlbumName}`

                        this.dirty = true;
                        this.scrollDisabled = false;

                        if (this.filter.order !== this.model.AlbumOrder) {
                            this.filter.order = this.model.AlbumOrder;
                            this.updateQuery();
                        } else {
                            this.loadMore();
                        }

                        return;
                    }
                }
            },
            onUpdate(ev, data) {
                if (!this.listen) return;

                if (!data || !data.entities) {
                    return
                }

                const type = ev.split('.')[1];

                switch (type) {
                    case 'updated':
                        for (let i = 0; i < data.entities.length; i++) {
                            const values = data.entities[i];
                            const model = this.results.find((m) => m.PhotoUUID === values.PhotoUUID);

                            if (model) {
                                for (let key in values) {
                                    if (values.hasOwnProperty(key) && values[key] != null && typeof values[key] !== "object") {
                                        model[key] = values[key];
                                    }
                                }
                            }
                        }
                        break;
                    case 'restored':
                        this.dirty = true;
                        this.scrollDisabled = false;

                        this.loadMore();

                        break;
                    case 'archived':
                        this.dirty = true;

                        for (let i = 0; i < data.entities.length; i++) {
                            const uuid = data.entities[i];
                            const index = this.results.findIndex((m) => m.PhotoUUID === uuid);
                            if (index >= 0) {
                                this.results.splice(index, 1);
                            }
                        }

                        break;
                }
            },
        },
        created() {
            this.findAlbum().then(() => this.search());

            this.subscriptions.push(Event.subscribe("albums.updated", (ev, data) => this.onAlbumsUpdated(ev, data)));
            this.subscriptions.push(Event.subscribe("photos", (ev, data) => this.onUpdate(ev, data)));

            this.subscriptions.push(Event.subscribe("touchmove.top", () => this.refresh()));
            this.subscriptions.push(Event.subscribe("touchmove.bottom", () => this.loadMore()));
        },
        destroyed() {
            for (let i = 0; i < this.subscriptions.length; i++) {
                Event.unsubscribe(this.subscriptions[i]);
            }
        },
    };
</script>

<template>
</template>

<script>
import _ from "lodash"

export default {
    inject: ["openmct"],
    props: {
        view: String,
        object: Object
    },
    destroyed() {
        this.clear();
    },
    watch: {
        view(newView, oldView) {
            this.viewKey = newView;
            this.debounceUpdateView();
        },
        object(newObject, oldObject) {
            this.currentObject = newObject;
            this.debounceUpdateView();
        }
    },
    created() {
        this.debounceUpdateView = _.debounce(this.updateView, 10);
    },
    mounted() {
        this.currentObject = this.object;
        this.updateView();
        this.$el.addEventListener('dragover', this.onDragOver);
        this.$el.addEventListener('drop', this.onDrop);
    },
    methods: {
        clear() {
            if (this.currentView) {
                this.currentView.destroy();
                this.$el.innerHTML = '';
            }
            delete this.viewContainer;
            delete this.currentView;
        },
        updateView(immediatelySelect) {
            this.clear();
            if (!this.currentObject) {
                return;
            }
            this.viewContainer = document.createElement('div');
            this.viewContainer.classList.add('c-object-view','u-contents');
            this.$el.append(this.viewContainer);
            let provider = this.openmct.objectViews.getByProviderKey(this.viewKey);
            if (!provider) {
                provider = this.openmct.objectViews.get(this.currentObject)[0];
                if (!provider) {
                    return;
                }
            }
            this.currentView = provider.view(this.currentObject);
            this.currentView.show(this.viewContainer);

            if (immediatelySelect) {
                this.removeSelectable = openmct.selection.selectable(
                    this.$el,
                    this.currentView.getSelectionContext ?
                        this.currentView.getSelectionContext() :
                        { item: this.currentObject },
                    true
                );
            }
        },
        show(object, viewKey, immediatelySelect) {
            if (this.unlisten) {
                this.unlisten();
            }

            if (this.removeSelectable) {
                this.removeSelectable();
                delete this.removeSelectable;
            }

            this.currentObject = object;
            this.unlisten = this.openmct.objects.observe(this.currentObject, '*', (mutatedObject) => {
                this.currentObject = mutatedObject;
            });

            this.viewKey = viewKey;
            this.updateView(immediatelySelect);
        },
        onDragOver(event) {
            if (this.hasComposableDomainObject(event)) {
                event.preventDefault();
            }
        },
        onDrop(event) {
            if (this.hasComposableDomainObject(event)) {
                let composableDomainObject = this.getComposableDomainObject(event);
                this.currentObject.composition.push(composableDomainObject.identifier);
                this.openmct.objects.mutate(this.currentObject, 'composition', this.currentObject.composition);
                event.preventDefault();
                event.stopPropagation();
            }
        },
        hasComposableDomainObject(event) {
            return event.dataTransfer.types.includes('openmct/composable-domain-object')
        },
        getComposableDomainObject(event) {
            let serializedDomainObject = event.dataTransfer.getData('openmct/composable-domain-object');
            return JSON.parse(serializedDomainObject);
        }
    }
}
</script>


<script>
    import ApiClient from "@/utils/ApiClient";
    import CommonHelper from "@/utils/CommonHelper";
    import ObjectSelect from "@/components/base/ObjectSelect.svelte";
    import RecordSelectOption from "./RecordSelectOption.svelte";

    const uniqueId = "select_" + CommonHelper.randomString(5);

    // original select props
    export let multiple = false;
    export let selected = multiple ? [] : undefined;
    export let keyOfSelected = multiple ? [] : undefined;
    export let selectPlaceholder = "- Select -";
    export let optionComponent = RecordSelectOption; // custom component to use for each dropdown option item

    // custom props
    export let collectionId;

    let list = [];
    let currentPage = 1;
    let totalItems = 0;
    let isLoadingList = false;
    let isLoadingSelected = false;

    $: if (collectionId) {
        loadList();
    }

    $: isLoading = isLoadingList || isLoadingSelected;

    $: canLoadMore = totalItems > list.length;

    loadSelected();

    async function loadSelected() {
        const selectedIds = CommonHelper.toArray(keyOfSelected);

        if (!collectionId || !selectedIds.length) {
            return;
        }

        isLoadingSelected = true;

        try {
            const filters = [];
            for (const id of selectedIds) {
                filters.push(`id="${id}"`);
            }

            selected = await ApiClient.Records.getFullList(collectionId, 200, {
                sort: "-created",
                filter: filters.join("||"),
                $cancelKey: uniqueId + "loadSelected",
            });

            // add the selected models to the list (if not already)
            list = CommonHelper.filterDuplicatesByKey(list.concat(selected));
        } catch (err) {
            ApiClient.errorResponseHandler(err);
        }

        isLoadingSelected = false;
    }

    async function loadList(reset = false) {
        if (!collectionId) {
            return;
        }

        isLoadingList = true;

        try {
            const page = reset ? 1 : currentPage + 1;

            const result = await ApiClient.Records.getList(collectionId, page, 200, {
                sort: "-created",
                $cancelKey: uniqueId + "loadList",
            });

            if (reset) {
                list = [];
            }

            list = CommonHelper.filterDuplicatesByKey(list.concat(result.items));
            currentPage = result.page;
            totalItems = result.totalItems;
        } catch (err) {
            ApiClient.errorResponseHandler(err);
        }

        isLoadingList = false;
    }
</script>

<ObjectSelect
    selectPlaceholder={isLoading ? "Loading..." : selectPlaceholder}
    items={list}
    searchable={list.length > 5}
    selectionKey="id"
    labelComponent={optionComponent}
    {optionComponent}
    {multiple}
    bind:keyOfSelected
    bind:selected
    on:show
    on:hide
    class="records-select block-options"
    {...$$restProps}
>
    <svelte:fragment slot="afterOptions">
        {#if canLoadMore}
            <button
                type="button"
                class="btn btn-block btn-sm"
                class:btn-loading={isLoadingList}
                class:btn-disabled={isLoadingList}
                on:click|stopPropagation={() => loadList()}
            >
                <span class="txt">Load more</span>
            </button>
        {/if}
    </svelte:fragment>
</ObjectSelect>

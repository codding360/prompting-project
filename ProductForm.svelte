<script lang="ts">
  import Input from "$lib/components/Input.svelte";
  import { storeState } from "$lib/store/store.store";
  import type { ProductType } from "$lib/types/product";
  import { artifactIntelligenceApi } from "$lib/api/artifact-intelligence";
  import { toast } from "svelte-sonner";
  import GalleryImageSelector from "$lib/components/GalleryImageSelector.svelte";

  let {
    data,
    isLoading,
    mode = "create",
    onupdateimages,
  }: {
    data: Partial<ProductType>;
    isLoading: boolean;
    mode?: "create" | "update";
    onupdateimages?: (publicUrlImage: string) => void;
  } = $props();


  let b2cOpen = $state(false);
  let generatingAI = $state(false);
  let b2cElement: HTMLDivElement | undefined = $state();
  let tagInput = $state("");

  // Memoized derived values for better performance
  const canGenerateDescription = $derived(!!data.title && !isLoading && !generatingAI && data.images?.[0]);

  async function generateDescription(e: Event) {
    e.preventDefault();
    e.stopPropagation();
    generatingAI = true;
    try {
      const response = await artifactIntelligenceApi.generateDescription(data.title!, data.images?.[0]!);
      if (response) {
        data.description = response.description;
      }
    } catch (error) {
      toast.error("Не удалось сгенерировать описание");
    } finally {
      generatingAI = false;
    }
  }

  // Scroll to accordion when opened
  function scrollToElement(element: HTMLDivElement | undefined) {
    if (element) {
      setTimeout(() => {
        element.scrollIntoView({ behavior: "smooth", block: "nearest" });
      }, 150);
    }
  }

  $effect(() => {
    if (b2cOpen && b2cElement) {
      scrollToElement(b2cElement);
    }
  });

  function handleRemoveImageId(publicUrlImage: string) {
    data.images = data.images?.filter(image => image !== publicUrlImage) ?? [];
  }

  function handleTagInput(e: KeyboardEvent) {
    const target = e.target as HTMLInputElement;
    const value = target.value.trim();
    
    // Add tag on space key
    if (e.key === " " && value) {
      e.preventDefault();
      addTag(value);
      tagInput = "";
    }
    // Add tag on Enter key
    else if (e.key === "Enter" && value) {
      e.preventDefault();
      addTag(value);
      tagInput = "";
    }
    // Remove last tag on Backspace if input is empty
    else if (e.key === "Backspace" && !target.value && data.tags?.length) {
      e.preventDefault();
      removeTag(data.tags.length - 1);
    }
  }

  function addTag(tag: string) {
    const trimmedTag = tag.trim();
    if (!trimmedTag) return;
    
    // Initialize tags array if it doesn't exist
    if (!data.tags) {
      data.tags = [];
    }
    
    // Don't add duplicate tags
    if (!data.tags.includes(trimmedTag)) {
      data.tags = [...data.tags, trimmedTag];
    }
  }

  function removeTag(index: number) {
    if (data.tags) {
      data.tags = data.tags.filter((_: string, i: number) => i !== index);
    }
  }
</script>

<form class="flex flex-col gap-3 sm:gap-4">
  <!-- Core Required Fields -->
  <div class="card bg-base-200">
    <div class="card-body p-3 sm:p-4">
      <!-- Images of product -->
      <fieldset class="fieldset">
        <legend class="fieldset-legend text-xs sm:text-sm">
          Изображения товара *
        </legend>

        <GalleryImageSelector
          images={data.images?.map(image => ({ id: image, preview: image }))}
          onchange={(newFiles) => onupdateimages?.(newFiles.map(file => URL.createObjectURL(file)).join(','))}
          onremove={handleRemoveImageId}
        />
      </fieldset>
      
      <!-- title of product -->
      <fieldset class="fieldset">
        <legend class="fieldset-legend text-xs sm:text-sm"
          >Название продукта *</legend
        >
        <Input
          type="text"
          placeholder="Например: iPhone 15 Pro"
          value={data.title}
          disabled={isLoading}
          oninput={(e) => {
            const target = e.target as HTMLInputElement;
            data.title = target.value;
          }}
          required
        />
      </fieldset>

      <!-- code of product -->
      <fieldset class="fieldset">
        <legend class="fieldset-legend text-xs sm:text-sm">Код продукта</legend>
        <div class="relative">
          <Input
            type="text"
            placeholder="Например: PROD-0001"
            value={data.code || ""}
            disabled={isLoading}
            oninput={(e) => {
              const target = e.target as HTMLInputElement;
              data.code = target.value;
            }}
          />
          {#if isLoading && !data.code && mode === "create"}
            <span class="absolute right-3 top-1/2 -translate-y-1/2">
              <span class="loading loading-spinner loading-xs"></span>
            </span>
          {/if}
        </div>
        {#if mode === "create"}
          <div class="label py-1">
            <span class="label-text-alt text-[10px] sm:text-xs text-base-content/60">
              Можете свой код писать или автоматически сгенеририруется
            </span>
          </div>
        {/if}
      </fieldset>

      <!-- Price of product -->
      <fieldset class="fieldset">
        <legend class="fieldset-legend text-xs sm:text-sm">Цена *</legend>
        <div class="relative">
          <Input
            type="number"
            placeholder="0.00"
            value={data.price?.toString() ?? undefined}
            min="0"
            step="0.01"
            disabled={isLoading}
            required
            oninput={(e) => {
              const target = e.target as HTMLInputElement;
              data.price = parseFloat(target.value);
            }}
          />
          <span
            class="absolute right-6 top-1/2 -translate-y-1/2 text-sm text-base-content/60 font-medium"
          >
            {$storeState?.currency || 'USD'}
          </span>
        </div>
      </fieldset>

      <!-- Category of product -->  
      <!-- <fieldset class="fieldset">
        <legend class="fieldset-legend text-xs sm:text-sm">Категория</legend>
        <CategorySelect 
          bind:value={data.categoryId} 
          disabled={isLoading} 
          onselect={(categoryId) => {
            data.categoryId = categoryId;
          }}
        />
      </fieldset> -->


    </div>
  </div>

  <!-- Additional fields -->
  <div bind:this={b2cElement} class="collapse collapse-arrow bg-base-200">
    <input type="checkbox" bind:checked={b2cOpen} />
    <div class="collapse-title font-semibold text-sm">
      📦 Доп. поля (необязательно)
    </div>
    <div class="collapse-content">
      <div class="flex flex-col gap-3 pt-2">
        <!-- Bulk price and MOQ -->
        <div
          class="grid grid-cols-2 gap-x-2 gap-y-1 bg-warning/10 p-2 rounded-lg"
        >
          <fieldset class="fieldset">
            <legend class="fieldset-legend text-xs sm:text-sm"
              >Специальная цена</legend
            >
            <div class="relative">
              <Input
                type="number"
                placeholder="0.00"
                value={data.bulkPrice?.toString() || ""}
                min="0"
                step="0.01"
                disabled={isLoading}
                oninput={(e: Event) => {
                  const target = e.target as HTMLInputElement;
                  data.bulkPrice = target.value ? parseFloat(target.value) : 0;
                }}
              />
              <span
                class="absolute right-6 top-1/2 -translate-y-1/2 text-sm text-base-content/60 font-medium"
              >
                {$storeState?.currency || 'USD'}
              </span>
            </div>
          </fieldset>
          <fieldset class="fieldset">
            <legend class="fieldset-legend text-xs sm:text-sm"
              >Мин. Кол. Заказ</legend
            >
            <Input
              type="number"
              placeholder="10"
              value={data.moq?.toString() || ""}
              min="1"
              oninput={(e: Event) => {
                const target = e.target as HTMLInputElement;
                data.moq = target.value ? parseInt(target.value) : 0;
              }}
              disabled={isLoading}
            />
          </fieldset>
        </div>
        <!-- Cost -->
        <fieldset class="fieldset">
          <legend class="fieldset-legend text-xs sm:text-sm">Себестоимость</legend>
          <div class="relative">
            <Input
              type="number"
              placeholder="0.00"
              value={data.cost?.toString() || ""}
              min="0"
              step="0.01"
              disabled={isLoading}
              oninput={(e) => {
                const target = e.target as HTMLInputElement;
                data.cost = target.value ? parseFloat(target.value) : 0;
              }}
            />
            <span
              class="absolute right-6 top-1/2 -translate-y-1/2 text-sm text-base-content/60 font-medium"
            >
              {$storeState?.currency || 'USD'}
            </span>
          </div>
          <div class="label py-1">
            <span
              class="label-text-alt text-[10px] sm:text-xs text-base-content/60"
            >
              Себесостоимость товара будет показана только вам
            </span>
          </div>
        </fieldset>

        <!-- Description -->
        <fieldset class="fieldset">
          <div class="flex items-center gap-2 justify-between">
            <legend class="fieldset-legend text-xs sm:text-sm">Описание</legend> 
            <button
              type="button"
              class="btn btn-sm btn-primary btn-outline gap-1"
              onclick={generateDescription}
              disabled={!canGenerateDescription}
              title="Генерировать описание с помощью AI"
            >
              {#if generatingAI}
                <span class="loading loading-spinner loading-xs"></span>
              {:else}
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"></path>
                </svg>
              {/if}
              <span>AI</span>
            </button>
          </div>
          <div class="flex flex-col gap-2 mt-2">
            <textarea
              class="textarea textarea-md textarea-bordered w-full focus:outline-none border border-base-300 focus:border-primary/40 shadow-none text-sm"
              placeholder="Подробное описание продукта"
              bind:value={data.description}
              disabled={isLoading}
              rows="4"
            ></textarea>
          </div>
          <div class="label py-1">
            <span
              class="label-text-alt text-[10px] sm:text-xs text-base-content/60"
            >
              Нажмите ⚡ для генерации описания через AI
            </span>
          </div>
        </fieldset>

        <fieldset class="fieldset">
          <legend class="fieldset-legend text-xs sm:text-sm">Теги</legend>
          <div class="flex flex-wrap gap-2 p-3 bg-base-100 border border-base-300 rounded-lg min-h-[3.5rem] focus-within:border-primary/40 transition-colors">
            {#if data.tags && data.tags.length > 0}
              {#each data.tags as tag, index}
                <div class="badge badge-primary gap-2 py-3 px-3 text-sm">
                  <span>{tag}</span>
                  <button
                    title="Удалить тег"
                    type="button"
                    class="btn btn-ghost btn-xs btn-circle h-4 w-4 min-h-0"
                    onclick={() => removeTag(index)}
                    disabled={isLoading}
                  >
                    <svg
                      xmlns="http://www.w3.org/2000/svg"
                      fill="none"
                      viewBox="0 0 24 24"
                      class="inline-block w-3 h-3 stroke-current"
                    >
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M6 18L18 6M6 6l12 12"
                      ></path>
                    </svg>
                  </button>
                </div>
              {/each}
            {/if}
            <input
              type="text"
              class="flex-1 min-w-[120px] bg-transparent outline-none text-base"
              placeholder={data.tags?.length ? "Добавить тег..." : "новинка скидка популярное"}
              bind:value={tagInput}
              onkeydown={handleTagInput}
              disabled={isLoading}
            />
          </div>
          <div class="label py-1">
            <span class="label-text-alt text-[10px] sm:text-xs text-base-content/60">
              Нажмите пробел или Enter для добавления тега
            </span>
          </div>
        </fieldset>
      </div>
    </div>
  </div>
</form>

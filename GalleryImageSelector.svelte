<script lang="ts">
  import type { HTMLInputAttributes } from "svelte/elements";

  type GalleryImage = {
    id: string;
    preview: string;
  };

  type GalleryImageSelectorProps = Omit<HTMLInputAttributes, 'onchange'> & {
    label?: string;
    description?: string;
    images?: GalleryImage[];
    maxSize?: number;
    maxImages?: number;
    acceptedFormats?: string[];
    required?: boolean;
    disabled?: boolean;
    loading?: boolean;
    onchange?: (files: File[]) => void;
    onerror?: (error: string) => void;
    onremove?: (id: string) => void;
  };

  let {
    label = "Галерея изображений",
    description = "PNG, JPG или WEBP (максимум 5MB каждое)",
    images = [],
    maxSize = 5,
    maxImages = 10,
    acceptedFormats = ["image/*"],
    required = false,
    disabled = false,
    loading = false,
    onchange,
    onerror,
    onremove,
    ...rest
  }: GalleryImageSelectorProps = $props();

  let isDragging = $state(false);
  let fileInput: HTMLInputElement;

  const canAddMore = $derived(images.length < maxImages);

  const handleFileChange = (e: Event) => {
    const target = e.target as HTMLInputElement;
    const files = Array.from(target.files || []);
    
    if (files.length === 0) return;

    // Check if adding these files would exceed the limit
    if (images.length + files.length > maxImages) {
      onerror?.(`Максимальное количество изображений: ${maxImages}`);
      return;
    }

    const validFiles = files.filter(file => validateFile(file));
    
    if (validFiles.length > 0) {
      onchange?.(validFiles);
    }
    
    // Reset input
    target.value = "";
  };

  const validateFile = (file: File): boolean => {
    const fileSizeMB = file.size / (1024 * 1024);
    if (fileSizeMB > maxSize) {
      onerror?.(`Файл ${file.name} слишком большой. Максимальный размер: ${maxSize}MB`);
      return false;
    }

    const fileType = file.type;
    const isValidType = acceptedFormats.some(format => {
      if (format === "image/*") return fileType.startsWith("image/");
      return fileType === format;
    });

    if (!isValidType) {
      onerror?.(`Файл ${file.name} имеет неподдерживаемый формат`);
      return false;
    }

    return true;
  };

  const handleDragOver = (e: DragEvent) => {
    e.preventDefault();
    if (!disabled && !loading && canAddMore) {
      isDragging = true;
    }
  };

  const handleDragLeave = (e: DragEvent) => {
    e.preventDefault();
    isDragging = false;
  };

  const handleDrop = (e: DragEvent) => {
    e.preventDefault();
    isDragging = false;

    if (disabled || loading || !canAddMore) return;

    const files = Array.from(e.dataTransfer?.files || []);
    
    if (files.length === 0) return;

    // Check if adding these files would exceed the limit
    if (images.length + files.length > maxImages) {
      onerror?.(`Максимальное количество изображений: ${maxImages}`);
      return;
    }

    const validFiles = files.filter(file => validateFile(file));
    
    if (validFiles.length > 0) {
      const dataTransfer = new DataTransfer();
      validFiles.forEach(file => dataTransfer.items.add(file));
      fileInput.files = dataTransfer.files;
      
      onchange?.(validFiles);
    }
  };

  const handleClick = () => {
    if (!disabled && !loading && canAddMore) {
      fileInput.click();
    }
  };

  const handleRemove = (id: string, e: Event) => {
    e.stopPropagation();
    onremove?.(id);
  };
</script>

<fieldset class="fieldset">
  <legend class="fieldset-legend">
    {label}
    {#if maxImages}
      <span class="text-sm font-normal text-base-content/60 ml-2">
        ({images.length}/{maxImages})
      </span>
    {/if}
  </legend>

  <!-- Gallery Grid -->
  {#if images.length > 0}
    <div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4 mb-4">
      {#each images as image (image.id)}
        <div class="relative group aspect-square rounded-lg overflow-hidden border-2 border-base-300 bg-base-100 transition-all duration-200 hover:border-primary/40">
          <img 
            src={image.preview} 
            alt="Изображение галереи" 
            class="w-full h-full object-cover"
          />
          
          {#if loading}
            <div class="absolute inset-0 bg-base-content/50 flex items-center justify-center">
              <span class="loading loading-spinner loading-md text-primary"></span>
            </div>
          {:else}
            <div class="absolute inset-0 bg-gradient-to-t from-base-content/80 to-transparent opacity-0 group-hover:opacity-100 transition-opacity duration-200 flex items-end justify-center pb-2">
              <button
                aria-label="add image"
                type="button"
                class="btn btn-sm btn-error btn-circle"
                onclick={(e) => handleRemove(image.id, e)}
                disabled={disabled || (required && images.length === 1)}
              >
                <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                </svg>
              </button>
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}

  <!-- Upload Zone -->
  {#if canAddMore}
    <div
      class="relative group"
      role="button"
      tabindex="0"
      ondragover={handleDragOver}
      ondragleave={handleDragLeave}
      ondrop={handleDrop}
      onclick={handleClick}
      onkeydown={(e) => e.key === 'Enter' && handleClick()}
    >
      <div
        class="border-2 border-dashed rounded-lg p-6 transition-all duration-200 cursor-pointer
               {isDragging ? 'border-primary bg-primary/5 scale-[1.02]' : 'border-base-300 hover:border-primary/40 hover:bg-base-200/50'}
               {disabled || loading ? 'opacity-50 cursor-not-allowed' : ''}"
      >
        {#if loading}
          <div class="flex flex-col items-center justify-center gap-3">
            <span class="loading loading-spinner loading-lg text-primary"></span>
            <p class="text-sm text-base-content/60">Загрузка...</p>
          </div>
        {:else}
          <div class="flex flex-col items-center justify-center gap-3">
            <div class="rounded-full bg-primary/10 p-3">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-primary" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" />
              </svg>
            </div>
            
            <div class="text-center">
              <p class="text-sm font-medium text-base-content mb-1">
                {isDragging ? 'Отпустите файлы' : images.length > 0 ? 'Добавить еще изображения' : 'Добавить изображения'}
              </p>
              <p class="text-xs text-base-content/60">
                {description}
              </p>
            </div>

            <button
              type="button"
              class="btn btn-primary btn-sm"
              disabled={disabled}
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
              </svg>
              Выбрать файлы
            </button>
          </div>
        {/if}
      </div>

      <input
        bind:this={fileInput}
        type="file"
        class="hidden"
        accept={acceptedFormats.join(",")}
        onchange={handleFileChange}
        multiple
        {required}
        {...rest}
      />
    </div>
  {:else}
    <div class="alert alert-info">
      <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" class="stroke-current shrink-0 w-6 h-6">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
      </svg>
      <span>Достигнут лимит: {maxImages} изображений</span>
    </div>
  {/if}
</fieldset>

<style>
  [role="button"]:focus,
  [role="button"]:focus-visible {
    outline: none;
    box-shadow: none;
  }
</style>
<template>
  <div>
    <v-form ref="domUrlForm" @submit.prevent="createRecipe">
      <div>
        <v-card-title class="headline">
          {{ $t('recipe.create-recipe-from-an-image') }}
        </v-card-title>
        <v-card-text>
          <p>{{ $t('recipe.create-recipe-from-an-image-description') }}</p>
          <v-container class="pa-0">
            <v-row>
              <v-col cols="auto" align-self="center">
                <AppButtonUpload class="ml-auto" url="none" file-name="images" accept="image/*"
                  :text="$t('recipe.upload-image')" :text-btn="false" :post="false" @uploaded="uploadImage" />
              </v-col>
              <v-spacer />
            </v-row>

            <div v-if="uploadedImages.length > 0" class="mt-3">
              <v-row>
                <v-col cols="12" class="pb-0">
                  <v-card-text class="pa-0">
                    <p class="mb-0">
                      {{ $t('recipe.crop-and-rotate-the-image') }}
                    </p>
                  </v-card-text>
                </v-col>
              </v-row>
              <v-row style="max-width: 600px;">
                <v-spacer />
                <v-col v-for="(imageUrl, index) in uploadedImagesPreviewUrls" :key="index" cols="12">
                  <v-row>
                    <v-col cols="auto" align-self="center">
                      <ImageCropper :img="imageUrl" cropper-height="100%" cropper-width="100%"
                        @save="(croppedImage) => updateUploadedImage(index, croppedImage)" />

                      <v-btn color="error" @click="() => clearImage(index)">
                        <v-icon start>
                          {{ $globals.icons.close }}
                        </v-icon>
                        {{ $t('recipe.remove-image') }}
                      </v-btn>
                    </v-col>
                  </v-row>
                </v-col>
                <v-spacer />
              </v-row>
            </div>
          </v-container>
        </v-card-text>
        <v-card-actions v-if="uploadedImages.length > 0">
          <div>
            <p style="width: 250px">
              <BaseButton rounded block type="submit" :loading="loading" />
            </p>
            <p>
              <v-checkbox v-model="shouldTranslate" hide-details :label="$t('recipe.should-translate-description')"
                :disabled="loading" />
            </p>
            <p v-if="loading" class="mb-0">
              {{ $t('recipe.please-wait-image-procesing') }}
            </p>
          </div>
        </v-card-actions>
      </div>
    </v-form>
  </div>
</template>

<script lang="ts">
import { useUserApi } from "~/composables/api";
import { alert } from "~/composables/use-toast";
import type { VForm } from "~/types/auto-forms";

export default defineNuxtComponent({
  setup() {
    const state = reactive({
      loading: false,
    });

    const i18n = useI18n();
    const api = useUserApi();
    const route = useRoute();
    const router = useRouter();
    const groupSlug = computed(() => route.params.groupSlug || "");

    const domUrlForm = ref<VForm | null>(null);
    const uploadedImages = ref<(Blob | File)[]>([]);
    const uploadedImageNames = ref<string[]>([]);
    const uploadedImagesPreviewUrls = ref<string[]>([]);
    const shouldTranslate = ref(true);

    function uploadImage(fileObject: File) {
      uploadedImages.value = [...uploadedImages.value, fileObject];
      uploadedImageNames.value = [...uploadedImageNames.value, fileObject.name];
      uploadedImagesPreviewUrls.value = [...uploadedImagesPreviewUrls.value, URL.createObjectURL(fileObject)];
    }

    function clearImage(index: number) {
      URL.revokeObjectURL(uploadedImagesPreviewUrls.value[index]);

      uploadedImages.value = uploadedImages.value.filter((_, i) => i !== index);
      uploadedImageNames.value = uploadedImageNames.value.filter((_, i) => i !== index);
      uploadedImagesPreviewUrls.value = uploadedImagesPreviewUrls.value.filter((_, i) => i !== index);
    }

    async function createRecipe() {
      if (uploadedImages.value.length === 0) {
        return;
      }

      state.loading = true;
      const translateLanguage = shouldTranslate.value ? i18n.locale : undefined;
      const { data, error } = await api.recipes.createOneFromImages(uploadedImages.value, translateLanguage?.value);
      if (error || !data) {
        alert.error(i18n.t("events.something-went-wrong"));
        state.loading = false;
      }
      else {
        router.push(`/g/${groupSlug.value}/r/${data}`);
      }
    }

    function updateUploadedImage(index: number, croppedImage: Blob) {
      uploadedImages.value[index] = croppedImage;
    }

    return {
      ...toRefs(state),
      domUrlForm,
      uploadedImages,
      uploadedImagesPreviewUrls,
      shouldTranslate,
      uploadImage,
      clearImage,
      createRecipe,
      updateUploadedImage,
    };
  },
});
</script>

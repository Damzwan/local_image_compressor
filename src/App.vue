<template>
  <div>
    <div class="w-full flex justify-center items-center flex-col px-12 pt-4">
      <p class="text-white text-2xl font-bold mb-2">Local Image Compressor - No Upload Required!</p>
      <p class="text-white text-lg">
        Unlike most online compressors that upload images to a server, this tool processes
        everything locally, ensuring quick and private compression even with a slow internet
        connection.
      </p>
    </div>

    <div class="flex flex-col justify-center items-center p-8">
      <input
        accept="image/*"
        type="file"
        ref="fileInput"
        class="file-input file-input-bordered w-full max-w-xs"
        @change="onFileChanged"
        multiple
      />
      <div
        class="mt-4 w-full max-w-lg border-2 border-dashed border-gray-500 p-8 text-center rounded-md cursor-pointer"
        @dragover.prevent
        @drop.prevent="handleDrop"
        @click="handleClick"
      >
        <p class="text-gray-500">Drag and drop files here, or click to select files</p>
        <div class="mt-4 flex flex-wrap overflow-y-auto max-h-48">
          <div
            v-for="(image, index) in images"
            :key="index"
            class="m-2 w-20 h-20 flex justify-center items-center border rounded-md overflow-hidden relative"
          >
            <button
              class="btn btn-square absolute right-0 top-0 w-[24px] h-[24px] z-10 min-h-0"
              @click.stop="removeImg(index)"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="h-5 w-5"
                fill="none"
                viewBox="0 0 24 24"
                stroke="red"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M6 18L18 6M6 6l12 12"
                />
              </svg>
            </button>

            <div class="w-full h-full">
              <img
                :src="image"
                alt="Preview"
                class="object-cover w-full h-full"
                v-if="image != ''"
              />
              <div class="w-full h-full object-cover flex justify-center items-center" v-else>
                <span class="loading loading-spinner loading-xs"></span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div
        v-if="compressionTasks.length > 0"
        class="mt-4 w-full max-w-lg border-2 border-full border-gray-500 px-4 py-2 text-center rounded-md relative overflow-y-auto overflow-x-hidden max-h-96"
      >
        <p class="text-white text-xl">Compressed Images</p>
        <hr class="my-2 border-gray-600" />

        <div v-for="(task, index) in compressionTasks" :key="index" class="w-full my-1">
          <div class="flex justify-between items-center">
            <div class="w-14 h-14 border rounded-md overflow-hidden">
              <img
                :src="task.preview"
                alt="Compressed Preview"
                class="object-cover w-full h-full"
              />
            </div>
            <div class="flex flex-col items-center">
              <div v-if="task.status === 'compressing'">
                <p>Compressing...</p>
              </div>
              <div v-else-if="task.status === 'done'">
                <button
                  @click="downloadImage(task.compressedFile as File)"
                  class="btn btn-primary mt-2 mr-3"
                >
                  Download
                </button>
                <button
                  @click="copyImage(task.compressedFile as File)"
                  class="btn btn-secondary mt-2"
                >
                  Copy
                </button>
              </div>
              <div v-else>
                <p>Error: {{ task.error }}</p>
              </div>
            </div>
          </div>
          <hr class="my-2 border-gray-600" />
        </div>
      </div>
      <button
        class="btn btn-secondary fixed left-4 bottom-4"
        @click="downloadAll"
        :disabled="tasksToBeCompleted > 0 || compressionTasks.length == 0"
      >
        <p class="text-white text-2xl">Download all</p>
      </button>
    </div>

    <button
      class="btn fixed right-4 bottom-4 btn-primary"
      :disabled="images.length == 0"
      @click="compress"
    >
      <p class="text-2xl text-white">Compress</p>
    </button>

    <div class="toast animate-in fade-in bottom-12" v-show="showToast">
      <div class="alert alert-success flex flex-col" v-if="!copyFailed">
        <span>Copied to clipboard</span>
        <img
          :src="copiedImageUrl"
          width="128"
          alt=""
          v-show="copiedImageUrl"
          class="cursor-pointer"
        />
      </div>

      <div class="alert alert-error" v-else>
        <span>Something went wrong</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import imageCompression from 'browser-image-compression'
import JSZip from 'jszip'
import { saveAs } from 'file-saver'

const fileInput = ref()
const images = ref<string[]>([])
const savedFiles = ref<File[]>([])
const showToast = ref(false)
const copyFailed = ref(false)

const copiedImageUrl = ref('')

const compressionTasks = ref<
  Array<{ preview: string; status: string; compressedFile?: File; error?: string }>
>([])
const tasksToBeCompleted = ref(0)

function handleClick() {
  fileInput.value.click()
}

function handleDrop(event: DragEvent) {
  const dt = event.dataTransfer
  if (dt && dt.files.length) {
    handleFiles(dt.files)
  }
}

function handleFiles(files: FileList) {
  for (let i = 0; i < files.length; i++) {
    const file = files[i]
    savedFiles.value.push(file)
    images.value.push('')

    if (file && file.type.startsWith('image/')) {
      const reader = new FileReader()
      reader.onload = (e) => {
        if (e.target && e.target.result) {
          const img = new Image()
          img.src = e.target.result as string
          img.onload = () => {
            const canvas = document.createElement('canvas')
            const ctx = canvas.getContext('2d')

            const maxSize = 200 // Max size for the preview (200x200 pixels)
            let width = img.width
            let height = img.height

            if (width > height) {
              if (width > maxSize) {
                height *= maxSize / width
                width = maxSize
              }
            } else {
              if (height > maxSize) {
                width *= maxSize / height
                height = maxSize
              }
            }

            canvas.width = width
            canvas.height = height

            if (ctx) {
              ctx.drawImage(img, 0, 0, width, height)
              const dataUrl = canvas.toDataURL('image/jpeg', 0.7) // Adjust quality as needed
              images.value[i] = dataUrl
            }
          }
        }
      }
      reader.readAsDataURL(file)
    }
  }
}

function onFileChanged(e: Event) {
  const target = e.target as HTMLInputElement
  if (target.files && target.files.length > 0) {
    handleFiles(target.files)
  }
}

function removeImg(i: number) {
  images.value.splice(i, 1)
  savedFiles.value.splice(i, 1)
}

async function compress() {
  compressionTasks.value = []
  const options = {
    maxSizeMB: 1,
    maxWidthOrHeight: 1920,
    useWebWorker: true
  }

  tasksToBeCompleted.value = savedFiles.value.length
  compressionTasks.value = savedFiles.value.map((file, index) => ({
    preview: images.value[index],
    status: 'compressing'
  }))

  images.value = []

  for (let i = 0; i < savedFiles.value.length; i++) {
    const file = savedFiles.value[i]
    try {
      imageCompression(file, options).then((compressedFile) => {
        compressionTasks.value[i].compressedFile = compressedFile
        compressionTasks.value[i].status = 'done'
        tasksToBeCompleted.value -= 1
      })
    } catch (error: any) {
      compressionTasks.value[i].status = 'error'
      compressionTasks.value[i].error = error.message
    }
  }

  savedFiles.value = []
}

function downloadImage(file: File) {
  const url = URL.createObjectURL(file)
  const a = document.createElement('a')
  a.href = url
  a.download = file.name
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

let timeout: any
async function copyImage(file: File) {
  clearTimeout(timeout)
  showToast.value = false

  const url = URL.createObjectURL(file)
  const img = new Image()
  img.src = url
  img.onload = async () => {
    const canvas = document.createElement('canvas')
    canvas.width = img.width
    canvas.height = img.height
    const ctx = canvas.getContext('2d')
    if (ctx) {
      ctx.drawImage(img, 0, 0)
      canvas.toBlob(async (blob) => {
        if (blob) {
          try {
            await navigator.clipboard.write([
              new ClipboardItem({
                [blob.type]: blob
              })
            ])
            copyFailed.value = false
            showToast.value = true
            copiedImageUrl.value = url
          } catch (err) {
            copyFailed.value = true
            showToast.value = true
            timeout = setTimeout(() => (showToast.value = false), 2000)
            console.error('Failed to copy image:', err)
          }
        }
      }, 'image/png')
    }
  }
}

function downloadAll() {
  const zip = new JSZip()
  compressionTasks.value.forEach((task, index) => {
    if (task.compressedFile) {
      zip.file(`image-${index + 1}.jpeg`, task.compressedFile)
    }
  })
  zip.generateAsync({ type: 'blob' }).then((content: any) => {
    saveAs(content, 'compressed-images.zip')
  })
}
</script>

<style scoped></style>

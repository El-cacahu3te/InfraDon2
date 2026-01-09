<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

interface Post {
  _id: string
  _rev?: string
  post_name: string
  post_content: string
  total_likes?: number
  comments?: string[]
  attributes?: {
    creation_date: string
  }
  _conflicts?: string[]
  _attachments?: {
    [key: string]: {
      content_type: string
      data?: any
      stub?: boolean
      length?: number
    }
  }
}

interface Reaction {
  _id: string
  _rev?: string
  user_id: string
  post_id: string
  isliked: boolean
  comments: string[]
}

// ============ VARIABLES ============
const documentName = ref('')
const documentContent = ref('')
const newPostFile = ref<File | null>(null)

const postsData = ref<Post[]>([])
const reactionsData = ref<Reaction[]>([])

const showSyncMessage = ref(false)
const isOffline = ref(false)

const selectedConflict = ref<Post | null>(null)
const otherVersions = ref<Post[]>([])

const searchTerm = ref('')
const newComment = ref('')

// Top likes pagination
const currentTopPage = ref(0)
const isTopMode = ref(false)

// Affichage des posts (10 premiers / tous)
const showAllPosts = ref(false)

// URLs locales pour les médias
const mediaUrls = ref<Record<string, string>>({})

// Affichage des commentaires (dernier / tous)
const showAllComments = ref<Record<string, boolean>>({})

// Edition de nom de post (par post_id)
const editNames = ref<Record<string, string>>({})

const postsDB = ref<PouchDB.Database<Post> | null>(null)
const reactionsDB = ref<PouchDB.Database<Reaction> | null>(null)

let postsSyncHandler: PouchDB.Replication.Sync<Post> | null = null
let reactionsSyncHandler: PouchDB.Replication.Sync<Reaction> | null = null

// ============ CONFIG ============
const COUCHDB_URL = 'http://localhost:5984'
const USERNAME = 'elia'
const PASSWORD = 'admin'
const POSTS_DB_NAME = 'post_elia_nicolo'
const REACTIONS_DB_NAME = 'reactions_elia_nicolo'

const REMOTE_POSTS_URL = `http://${USERNAME}:${PASSWORD}@${COUCHDB_URL.replace('http://', '')}/${POSTS_DB_NAME}`
const REMOTE_REACTIONS_URL = `http://${USERNAME}:${PASSWORD}@${COUCHDB_URL.replace('http://', '')}/${REACTIONS_DB_NAME}`
// ============ DEBUG INDEXES ============
const debugIndexes = async () => {
  if (!postsDB.value) return
  const idx = await (postsDB.value as any).getIndexes()
  console.log('📚 Indexes PouchDB :', idx)
}


// ============ INIT DB ============
const initDatabase = async () => {
  console.log('🔧 Initialisation des bases...')
  postsDB.value = new PouchDB<Post>('local_posts')
  reactionsDB.value = new PouchDB<Reaction>('local_reactions')
  console.log('✅ Bases locales créées')

  try {
    await postsDB.value!.createIndex({ index: { fields: ['post_name'] } })
    await postsDB.value!.createIndex({
      index: { fields: ['total_likes'] },
      name: 'idx_total_likes',
      ddoc: 'idx-total-likes'
    })
    console.log('✅ Index créés')
  } catch (err) {
    console.error('❌ Erreur création index:', err)
  }

  await debugIndexes() // 🔍 ICI

  await checkCouchDBConnection()
  await performInitialSync()
  await fetchData()
  startLiveSync()
}

/*const initDatabase = async () => {
  console.log('🔧 Initialisation des bases...')
  postsDB.value = new PouchDB<Post>('local_posts')
  reactionsDB.value = new PouchDB<Reaction>('local_reactions')
  console.log('✅ Bases locales créées')

  try {
    await postsDB.value!.createIndex({ index: { fields: ['post_name'] } })
    await postsDB.value!.createIndex({ index: { fields: ['total_likes'] } })
    console.log('✅ Index créés')
  } catch (err) {
    console.error('❌ Erreur création index:', err)
  }

  await checkCouchDBConnection()
  await performInitialSync()
  await fetchData()
  startLiveSync()
} */

// ============ CHECK COUCHDB ============
const checkCouchDBConnection = async () => {
  console.log('🔍 Test connexion CouchDB...')

  try {
    const response = await fetch(COUCHDB_URL)
    const data = await response.json()
    console.log('✅ CouchDB répond:', data)
  } catch (err) {
    console.error('❌ CouchDB ne répond pas:', err)
    throw new Error('Impossible de se connecter à CouchDB')
  }

  try {
    const authHeader = 'Basic ' + btoa(`${USERNAME}:${PASSWORD}`)
    const response = await fetch(`${COUCHDB_URL}/_all_dbs`, {
      headers: { Authorization: authHeader }
    })
    const dbs = await response.json()

    if (!dbs.includes(POSTS_DB_NAME)) {
      throw new Error(`Base ${POSTS_DB_NAME} manquante`)
    }
    if (!dbs.includes(REACTIONS_DB_NAME)) {
      throw new Error(`Base ${REACTIONS_DB_NAME} manquante`)
    }
    console.log('✅ Les deux bases existent')
  } catch (err) {
    console.error('❌ Erreur vérification bases:', err)
    throw err
  }
}

// ============ SYNC INIT ============
const performInitialSync = async () => {
  if (!postsDB.value || !reactionsDB.value) return
  console.log('🔄 Synchronisation initiale...')

  try {
    await postsDB.value.replicate.from(REMOTE_POSTS_URL)
    await reactionsDB.value.replicate.from(REMOTE_REACTIONS_URL)
    console.log('✅ Synchronisation initiale terminée')
  } catch (err) {
    console.error('❌ Erreur synchro initiale:', err)
  }
}

// ============ LIVE SYNC ============
const startLiveSync = () => {
  if (!postsDB.value || !reactionsDB.value) return
  if (isOffline.value) return

  console.log('▶️ Démarrage de la sync live')

  postsSyncHandler = postsDB.value.sync(REMOTE_POSTS_URL, {
    live: true,
    retry: true
  })
    .on('change', () => {
      console.log('📥 Changement détecté (posts), refresh')
      fetchData()
    })
    .on('error', (err: any) => {
      console.error('❌ Erreur sync posts:', err)
    })

  reactionsSyncHandler = reactionsDB.value.sync(REMOTE_REACTIONS_URL, {
    live: true,
    retry: true
  })
    .on('change', () => {
      console.log('📥 Changement détecté (reactions), refresh')
      fetchData()
    })
    .on('error', (err: any) => {
      console.error('❌ Erreur sync reactions:', err)
    })
}

const stopLiveSync = () => {
  console.log('⏹️ Arrêt de la sync live')
  if (postsSyncHandler) postsSyncHandler.cancel()
  if (reactionsSyncHandler) reactionsSyncHandler.cancel()
  postsSyncHandler = null
  reactionsSyncHandler = null
}

// ============ FETCH DATA ============
const refreshEditNames = () => {
  const map: Record<string, string> = {}
  for (const p of postsData.value) {
    map[p._id] = p.post_name
  }
  editNames.value = map
}

const fetchData = async () => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    const postsResult = await postsDB.value.allDocs({
  include_docs: true,
  conflicts : true,
  attachments: true   // <-- important
})

    let posts = postsResult.rows
      .map(r => r.doc)
      .filter((d): d is Post => !!d)

    // Par défaut : seulement les 10 premiers
    if (!showAllPosts.value && !isTopMode.value) {
      posts = posts.slice(0, 10)
    }

    postsData.value = posts
        // === Génération des URLs pour les médias existants ===
    for (const p of postsData.value) {
      if (p._attachments && p._attachments['media'] && !mediaUrls.value[p._id]) {
        // On ne met pas `await` ici pour ne pas bloquer tout fetchData
        loadMedia(p._id)
      }
    }

    refreshEditNames()

    const reactionsResult = await reactionsDB.value.allDocs({ include_docs: true })
    reactionsData.value = reactionsResult.rows
      .map(r => r.doc)
      .filter((d): d is Reaction => !!d)

  } catch (err) {
    console.error('❌ Erreur fetchData:', err)
  }
}

// ============ MODE EN LIGNE / HORS LIGNE ============
const manualSync = async () => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    console.log('🔄 Sync manuelle...')
    await postsDB.value.replicate.to(REMOTE_POSTS_URL)
    await reactionsDB.value.replicate.to(REMOTE_REACTIONS_URL)
    showSyncMessage.value = true
    setTimeout(() => { showSyncMessage.value = false }, 2000)
  } catch (err) {
    console.error('❌ Erreur sync manuelle:', err)
  }
}

const toggleOffline = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) {
    stopLiveSync()
  } else {
    startLiveSync()
  }
}

// ============ CRUD POSTS ============

const onNewPostFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    newPostFile.value = input.files[0]
  } else {
    newPostFile.value = null
  }
}

const addDocument = async () => {
  if (!postsDB.value) return
  if (!documentName.value.trim()) return

  const id = `post_${Date.now()}_${Math.random().toString(16).slice(2)}`
  const newPost: Post = {
    _id: id,
    post_name: documentName.value.trim(),
    post_content: documentContent.value.trim(),
    total_likes: 0,
    comments: [],
    attributes: {
      creation_date: new Date().toISOString()
    }
  }

  try {
    const res = await postsDB.value.put(newPost)
    let rev = res.rev

    // Si un fichier est sélectionné, on l'attache tout de suite
    if (newPostFile.value) {
      const file = newPostFile.value
      const attachRes = await postsDB.value.putAttachment(
        id,
        'media',
        rev,
        file,
        file.type || 'application/octet-stream'
      )
      rev = attachRes.rev

      // URL locale pour prévisualisation
      mediaUrls.value[id] = URL.createObjectURL(file)
    }

    documentName.value = ''
    documentContent.value = ''
    newPostFile.value = null

    await fetchData()
  } catch (err) {
    console.error('❌ Erreur addDocument:', err)
  }
}

const savePostName = async (post: Post) => {
  if (!postsDB.value) return
  const newName = editNames.value[post._id]?.trim()
  if (!newName || newName === post.post_name) return

  try {
    const freshPost = await postsDB.value.get(post._id)
    freshPost.post_name = newName
    await postsDB.value.put(freshPost)
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur savePostName:', err)
  }
}

const cancelEditPostName = (post: Post) => {
  editNames.value[post._id] = post.post_name
}

const deleteDocument = async (id: string, rev?: string) => {
  if (!postsDB.value || !id || !rev) return

  try {
    await postsDB.value.remove(id, rev)
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur deleteDocument:', err)
  }
}

// ============ RECHERCHE ============
const searchPosts = async (term: string) => {
  if (!postsDB.value) return

  const trimmed = term.trim()
  if (!trimmed) {
    isTopMode.value = false
    await fetchData()
    return
  }

  try {
    const result = await postsDB.value.find({
      selector: {
        post_name: {
          $gte: trimmed,
          $lte: trimmed + '\uffff'
        }
      }
    })
    let docs = result.docs as Post[]
    if (!showAllPosts.value) {
      docs = docs.slice(0, 10)
    }
    postsData.value = docs
    isTopMode.value = false
    refreshEditNames()
  } catch (err) {
    console.error('❌ Erreur searchPosts:', err)
  }
}

// ============ REACTIONS / COMMENTAIRES ============

const getReactionForPost = (postId: string): Reaction | null => {
  return (
    reactionsData.value.find(
      r => r.post_id === postId && r.user_id === 'user_1'
    ) || null
  )
}

const fetchReactionsOnly = async () => {
  if (!reactionsDB.value) return
  const reactionsResult = await reactionsDB.value.allDocs({ include_docs: true })
  reactionsData.value = reactionsResult.rows
    .map(r => r.doc)
    .filter((d): d is Reaction => !!d)
}

const updateTotalLikes = async (postId: string) => {
  if (!reactionsDB.value || !postsDB.value) return

  try {
    const likeResult = await reactionsDB.value.find({
      selector: {
        post_id: postId,
        isliked: true
      },
      fields: ['_id']
    })
    const likes = likeResult.docs.length

    const post = await postsDB.value.get(postId)
    post.total_likes = likes
    await postsDB.value.put(post)
  } catch (err) {
    console.error('❌ Erreur updateTotalLikes:', err)
  }
}

const addReaction = async (postId: string, comment?: string, isLiked?: boolean) => {
  if (!reactionsDB.value) return

  try {
    const existing = getReactionForPost(postId)

    let reaction: Reaction
    if (existing) {
      // Recharger depuis la DB pour être sûr d'avoir la dernière _rev
      const fresh = await reactionsDB.value.get(existing._id)
      reaction = fresh
    } else {
      reaction = {
        _id: `reaction_${postId}_user_1`,
        user_id: 'user_1',
        post_id: postId,
        isliked: false,
        comments: []
      }
    }

    if (typeof isLiked === 'boolean') {
      reaction.isliked = isLiked
    }

    if (comment && comment.trim()) {
      reaction.comments.push(comment.trim())
    }

    await reactionsDB.value.put(reaction)
    newComment.value = ''
    await fetchReactionsOnly()
    await updateTotalLikes(postId)
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur addReaction:', err)
  }
}

const deleteComment = async (postId: string, comment: string) => {
  if (!reactionsDB.value) return
  const existing = getReactionForPost(postId)
  if (!existing) return

  try {
    const fresh = await reactionsDB.value.get(existing._id)
    fresh.comments = fresh.comments.filter(c => c !== comment)
    await reactionsDB.value.put(fresh)
    await fetchReactionsOnly()
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur deleteComment:', err)
  }
}

const editComment = async (postId: string, oldComment: string) => {
  if (!reactionsDB.value) return
  const existing = getReactionForPost(postId)
  if (!existing) return

  const newText = window.prompt('Modifier le commentaire :', oldComment)
  if (newText === null) return
  const trimmed = newText.trim()
  if (!trimmed) return

  try {
    const fresh = await reactionsDB.value.get(existing._id)
    const idx = fresh.comments.indexOf(oldComment)
    if (idx !== -1) {
      fresh.comments[idx] = trimmed
      await reactionsDB.value.put(fresh)
      await fetchReactionsOnly()
      await fetchData()
    }
  } catch (err) {
    console.error('❌ Erreur editComment:', err)
  }
}

const toggleCommentsView = (postId: string) => {
  showAllComments.value[postId] = !showAllComments.value[postId]
}

// ============ FACTORY ============
const generateFactoryData = async () => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    // coupe la sync live pendant la génération si online
    const wasOffline = isOffline.value
    if (!wasOffline) {
      stopLiveSync()
    }

    const posts: Post[] = []
    const reactions: Reaction[] = []

    for (let i = 0; i < 50; i++) {
      const id = `factory_post_${Date.now()}_${i}_${Math.random().toString(16).slice(2)}`
      const randomLikes = Math.floor(Math.random() * 51)  

      posts.push({
        _id: id,
        post_name: `Post factory #${i + 1}`,
        post_content: `Contenu généré automatiquement pour le post #${i + 1}`,
        total_likes: randomLikes,
        comments: [],
        attributes: {
          creation_date: new Date().toISOString()
        }
      })

      for (let j = 0; j < randomLikes; j++) {
        reactions.push({
          _id: `reaction_${id}_user_${j}`,
          user_id: `user_${j}`,
          post_id: id,
          isliked: true,
          comments: []
        })
      }
    }

    await postsDB.value.bulkDocs(posts)
    await reactionsDB.value.bulkDocs(reactions)

    await fetchReactionsOnly()
    await fetchData()

    // relance la sync live si elle était active au départ
    if (!wasOffline) {
      startLiveSync()
    }
  } catch (err) {
    console.error('❌ Erreur generateFactoryData:', err)
  }
}

// ============ TOP LIKES ============
const getTopLikedPosts = async (page = 0) => {
  if (!postsDB.value) return

  try {
    const result = await postsDB.value.find({
      selector: {
        total_likes: { $gte: 0 }
      },
      sort: [{ total_likes: 'desc' as any }],
      limit: 10,
      skip: page * 10
    })

    postsData.value = result.docs as Post[]
    isTopMode.value = true
    showAllPosts.value = true // en mode top, on ne limite pas à 10 par défaut
    currentTopPage.value = page
    refreshEditNames()
  } catch (err) {
    console.error('❌ Erreur getTopLikedPosts:', err)
  }
}

const prevTopPage = () => {
  if (currentTopPage.value === 0) return
  getTopLikedPosts(currentTopPage.value - 1)
}

const nextTopPage = () => {
  getTopLikedPosts(currentTopPage.value + 1)
}

// ============ ATTACHMENTS (MÉDIAS) ============
const addMedia = async (post: Post, event: Event) => {
  if (!postsDB.value) return
  const input = event.target as HTMLInputElement
  if (!input.files || !input.files[0]) return

  const file = input.files[0]

  try {
    const freshPost = await postsDB.value.get(post._id)
    const res = await postsDB.value.putAttachment(
      freshPost._id,
      'media',
      freshPost._rev!,
      file,
      file.type || 'application/octet-stream'
    )

    // mettre à jour la rev localement
    post._rev = res.rev
    mediaUrls.value[post._id] = URL.createObjectURL(file)

    await fetchData()
  } catch (err) {
    console.error('❌ Erreur addMedia:', err)
  } finally {
    input.value = ''
  }
}

const loadMedia = async (postId: string) => {
  if (!postsDB.value) return

  try {
    const blob = await postsDB.value.getAttachment(postId, 'media')
    const url = URL.createObjectURL(blob as Blob)
    mediaUrls.value[postId] = url
  } catch (err) {
    console.error('❌ Erreur loadMedia:', err)
  }
}

const removeMedia = async (post: Post) => {
  if (!postsDB.value || !post._rev) return

  try {
    const res = await postsDB.value.removeAttachment(post._id, 'media', post._rev)
    post._rev = res.rev
    delete mediaUrls.value[post._id]
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur removeMedia:', err)
  }
}

// ============ CONFLITS ============
const resolveConflict = async (postId: string) => {
  if (!postsDB.value) return

  try {
    const doc = await postsDB.value.get(postId, { conflicts: true })
    selectedConflict.value = doc

    const conflicts = doc._conflicts || []
    const versions: Post[] = []

    for (const rev of conflicts) {
      const v = await postsDB.value.get(postId, { rev })
      versions.push(v)
    }
    otherVersions.value = versions
  } catch (err) {
    console.error('❌ Erreur resolveConflict:', err)
  }
}

const keepLocal = async () => {
  if (!postsDB.value || !selectedConflict.value) return

  try {
    const doc = selectedConflict.value
    const conflicts = doc._conflicts || []

    for (const rev of conflicts) {
      const conflictDoc = await postsDB.value.get(doc._id, { rev })
      await postsDB.value.remove(conflictDoc._id, conflictDoc._rev!)
    }

    selectedConflict.value = null
    otherVersions.value = []
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur keepLocal:', err)
  }
}

const keepRemote = async (idx: number) => {
  if (!postsDB.value || !selectedConflict.value) return

  try {
    const chosen = otherVersions.value[idx]
    const current = await postsDB.value.get(selectedConflict.value._id)

    chosen._rev = current._rev
    await postsDB.value.put(chosen)

    const conflicts = current._conflicts || []
    for (const rev of conflicts) {
      if (rev !== chosen._rev) {
        const conflictDoc = await postsDB.value.get(current._id, { rev })
        await postsDB.value.remove(conflictDoc._id, conflictDoc._rev!)
      }
    }

    selectedConflict.value = null
    otherVersions.value = []
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur keepRemote:', err)
  }
}

const cancelConflictResolution = () => {
  selectedConflict.value = null
  otherVersions.value = []
}

// ============ LIFECYCLE ============
onMounted(async () => {
  await initDatabase()
})

onUnmounted(() => {
  stopLiveSync()
})
</script>
<template>
  <div style="padding: 20px; max-width: 800px; margin: 0 auto;">
    <h1>Gestion Posts & Réactions</h1>

    <!-- ZONE BOUTONS + MESSAGES -->
    <div style="background: #f5f5f5; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
      <button @click="manualSync" :disabled="!isOffline" :style="{ opacity: isOffline ? 1 : 0.5 }">
        🔄 Synchroniser manuellement
      </button>

      <button
        @click="toggleOffline"
        :style="{ background: isOffline ? '#f44336' : '#4caf50', color: 'white' }"
      >
        {{ isOffline ? '🔴 Hors ligne' : '🟢 En ligne' }}
      </button>

      <button @click="fetchData">🔥 Rafraîchir</button>

      <button @click="getTopLikedPosts()">⭐ Top likes</button>

      <button @click="generateFactoryData()">🧪 Générer 50 posts (factory)</button>

      <div v-if="showSyncMessage" style="color: green; margin-top: 10px;">
        ✅ Synchronisation réussie
      </div>

      <div
        v-if="isOffline"
        style="background: #fff3cd; color: #856404; padding: 10px; margin-top: 10px; border-radius: 4px; font-weight: bold;"
      >
        🔴 MODE HORS LIGNE<br />
        <span style="font-weight: normal; font-size: 0.9em;">
          ⚠️ Sync automatique désactivée. Cliquez sur "Synchroniser manuellement" pour envoyer vos modifications.
        </span>
      </div>

      <div
        v-if="!isOffline"
        style="background: #d4edda; color: #155724; padding: 10px; margin-top: 10px; border-radius: 4px; font-weight: bold;"
      >
        🟢 MODE EN LIGNE<br />
        <span style="font-weight: normal; font-size: 0.9em;">
          ✅ Synchronisation automatique active. Vos modifications sont envoyées en temps réel.
        </span>
      </div>
    </div>

    <!-- SEARCH -->
    <div style="margin-bottom: 20px;">
      <input
        v-model="searchTerm"
        @input="searchPosts(searchTerm)"
        placeholder="🔍 Rechercher un post..."
        style="width: 100%; padding: 10px; font-size: 16px;"
      />
    </div>

    <!-- FORM NOUVEAU POST -->
    <div style="background: #e3f2fd; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
      <h3>➕ Nouveau post</h3>
      <input
        v-model="documentName"
        placeholder="Nom du post"
        style="width: 100%; margin-bottom: 10px; padding: 8px;"
      />
      <textarea
        v-model="documentContent"
        placeholder="Contenu du post"
        rows="3"
        style="width: 100%; margin-bottom: 10px; padding: 8px;"
      ></textarea>

      <div style="margin-bottom: 10px;">
        <label><strong>📎 Média (optionnel) :</strong></label><br />
        <input type="file" @change="onNewPostFileChange" />
      </div>

      <button @click="addDocument" :disabled="!documentName.trim()">Créer le post</button>
    </div>

    <!-- BOUTON 10 PREMIERS / TOUS (hors mode TOP) -->
    <div v-if="!isTopMode && postsData.length > 0" style="margin-bottom: 10px;">
      <button
        v-if="!showAllPosts"
        @click="showAllPosts = true; fetchData()"
      >
        👀 Voir tous les posts
      </button>
      <button
        v-else
        @click="showAllPosts = false; fetchData()"
      >
        🔙 Voir seulement les 10 premiers posts
      </button>
    </div>

    <!-- TOP LIKES PAGINATION -->
    <div v-if="isTopMode" style="margin-bottom: 10px;">
      <strong>Vue Top likes (page {{ currentTopPage + 1 }})</strong>
      <button @click="prevTopPage" :disabled="currentTopPage === 0">⬅️ Page précédente</button>
      <button @click="nextTopPage">➡️ Page suivante</button>
      <button @click="isTopMode = false; fetchData()">🔙 Revenir à la vue normale</button>
    </div>

    <!-- CONFLICTS RESOLUTION -->
    <div v-if="selectedConflict" style="background: #fff3cd; padding: 20px; margin-bottom: 20px; border-radius: 8px;">
      <h2>⚠️ Résolution de conflit</h2>

      <div style="background: #d4edda; padding: 10px; margin: 10px 0; border-radius: 4px;">
        <h4>Version locale (actuelle)</h4>
        <p><strong>Nom:</strong> {{ selectedConflict.post_name }}</p>
        <p><strong>Contenu:</strong> {{ selectedConflict.post_content }}</p>
        <p><strong>Likes:</strong> {{ selectedConflict.total_likes }}</p>
        <button @click="keepLocal" style="background: #28a745; color: white;">Garder cette version</button>
      </div>

      <div
        v-for="(version, idx) in otherVersions"
        :key="idx"
        style="background: #fff0f0; padding: 10px; margin: 10px 0; border-radius: 4px;"
      >
        <h4>Version distante {{ idx + 1 }}</h4>
        <p><strong>Nom:</strong> {{ version.post_name }}</p>
        <p><strong>Contenu:</strong> {{ version.post_content }}</p>
        <p><strong>Likes:</strong> {{ version.total_likes }}</p>
        <button @click="keepRemote(idx)" style="background: #dc3545; color: white;">Garder cette version</button>
      </div>

      <button
        @click="cancelConflictResolution"
        style="margin-top: 10px; background: #6c757d; color: white;"
      >
        Annuler
      </button>
    </div>

    <!-- POSTS LIST -->
    <div v-if="!selectedConflict">
      <h2>📄 Posts ({{ postsData.length }})</h2>

      <article
        v-for="post in postsData"
        :key="post._id"
        style="border: 1px solid #ccc; padding: 15px; margin: 15px 0; border-radius: 8px;"
      >
        <!-- CONFLICT WARNING -->
        <div
          v-if="post._conflicts && post._conflicts.length > 0"
          style="background: #ffebee; padding: 10px; margin-bottom: 15px; border-radius: 4px;"
        >
          <span style="color:red; font-weight: bold;">
            ⚠️ Conflit détecté ({{ post._conflicts.length }} version(s))
          </span>
          <button
            @click="resolveConflict(post._id)"
            style="margin-left: 10px; background: #ff9800; color: white;"
          >
            Résoudre le conflit
          </button>
        </div>

        <!-- POST CONTENT -->
        <h3>{{ post.post_name }}</h3>
        <p>{{ post.post_content }}</p>
        <p style="font-size: 0.85em; color: #666;">
          💾 ID: {{ post._id }} | 👍 {{ post.total_likes || 0 }} likes
        </p>

        <!-- MÉDIA -->
        <div style="margin-top: 10px; padding: 10px; border: 1px dashed #ccc; border-radius: 4px;">
          <p><strong>📎 Média associé :</strong></p>

          <div v-if="post._attachments && post._attachments['media']">
            <p>Média présent dans le document.</p>
            <div v-if="mediaUrls[post._id]">
              <a :href="mediaUrls[post._id]" target="_blank">Ouvrir le média</a>
            </div>
            
            <button @click="removeMedia(post)" style="background: #f44336; color: white;">
              🗑️ Supprimer le média
            </button>
          </div>

          <div v-else>
            <p>Aucun média associé.</p>
          </div>

          <div style="margin-top: 5px;">
            <input type="file" @change="(e) => addMedia(post, e)" />
          </div>
        </div>

        <!-- REACTIONS + COMMENTAIRES -->
        <div
          v-if="getReactionForPost(post._id)"
          style="background: #f9f9f9; padding: 10px; margin: 10px 0; border-radius: 4px;"
        >
          <p
            v-if="getReactionForPost(post._id)!.isliked"
            style="margin: 0 0 10px 0; color: #4caf50; font-weight: bold;"
          >
            ✅ Vous aimez ce post
          </p>
          <p v-else style="margin: 0 0 10px 0; color: #999;">
            Vous n'avez pas liké
          </p>

          <div v-if="getReactionForPost(post._id)!.comments.length > 0">
            <p style="margin: 10px 0 5px 0;">
              <strong>💬 Commentaires ({{ getReactionForPost(post._id)!.comments.length }}) :</strong>
              <button @click="toggleCommentsView(post._id)" style="margin-left: 10px;">
                {{ showAllComments[post._id] ? 'Voir seulement le dernier' : 'Voir tous les commentaires' }}
              </button>
            </p>
            <ul style="list-style: none; padding-left: 0; margin: 0;">
              <li
                v-for="(c, idx) in (showAllComments[post._id]
                  ? getReactionForPost(post._id)!.comments
                  : [getReactionForPost(post._id)!.comments[getReactionForPost(post._id)!.comments.length - 1]])"
                :key="idx"
                style="margin: 5px 0; padding: 8px; background: #fff; border: 1px solid #ddd; border-radius: 4px; display: flex; justify-content: space-between; align-items: center;"
              >
                <span>{{ c }}</span>
                <div>
                  <button
                    @click="editComment(post._id, c)"
                    style="background: #2196f3; color: white; padding: 4px 8px; font-size: 0.85em; border: none; border-radius: 3px; cursor: pointer; margin-right: 5px;"
                  >
                    ✏️
                  </button>
                  <button
                    @click="deleteComment(post._id, c)"
                    style="background: #f44336; color: white; padding: 4px 8px; font-size: 0.85em; border: none; border-radius: 3px; cursor: pointer;"
                  >
                    🗑️
                  </button>
                </div>
              </li>
            </ul>
          </div>
        </div>

        <!-- REACTION ACTIONS -->
        <div style="background: #f0f0f0; padding: 10px; margin: 10px 0; border-radius: 4px;">
          <button @click="addReaction(post._id, undefined, true)">👍 Like</button>
          <button @click="addReaction(post._id, undefined, false)">👎 Unlike</button>
          <input
            v-model="newComment"
            placeholder="Ajouter un commentaire"
            style="width: 60%; padding: 8px;"
          />
          <button @click="addReaction(post._id, newComment)" :disabled="!newComment.trim()">
            💬 Commenter
          </button>
        </div>

        <!-- POST ACTIONS -->
        <div style="margin-top: 15px; padding-top: 10px; border-top: 1px solid #ddd;">
          <input
            v-model="editNames[post._id]"
            placeholder="Nouveau nom"
            style="width: 60%; padding: 8px; margin-right: 5px;"
          />
          <button @click="savePostName(post)" style="background: #2196f3; color: white;">
            💾 Enregistrer le nom
          </button>
          <button @click="cancelEditPostName(post)" style="margin-left: 5px;">
            ❌ Annuler
          </button>
          <button @click="deleteDocument(post._id, post._rev)" style="background: #f44336; color: white; margin-left: 10px;">
            🗑️ Supprimer
          </button>
        </div>
      </article>
    </div>
  </div>
</template>



<style scoped>
button {
  margin: 5px;
  padding: 8px 12px;
  cursor: pointer;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: #fff;
  transition: background 0.2s;
}

button:hover {
  background: #e0e0e0;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

input,
textarea {
  margin: 5px;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

article {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
</style>
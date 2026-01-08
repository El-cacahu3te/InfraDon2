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
}

interface Reaction {
  _id: string
  _rev?: string
  user_id: string
  post_id: string
  isliked: boolean
  comments: string[]
}

// Variables réactives
const documentName = ref('')
const documentContent = ref('')
const postsData = ref<Post[]>([])
const showSyncMessage = ref(false)
const isOffline = ref(false)
const selectedConflict = ref<Post | null>(null)
const otherVersions = ref<Post[]>([])
const searchTerm = ref('')
const newComment = ref('')
const reactionsData = ref<Reaction[]>([])

const postsDB = ref<PouchDB.Database<Post>>()
const reactionsDB = ref<PouchDB.Database<Reaction>>()

// Handlers de sync à nettoyer
let postsSyncHandler: PouchDB.Replication.Sync<Post> | null = null
let reactionsSyncHandler: PouchDB.Replication.Sync<Reaction> | null = null

// Configuration
const COUCHDB_URL = 'http://localhost:5984'
const USERNAME = 'elia'
const PASSWORD = 'admin'
const POSTS_DB_NAME = 'post_elia_nicolo'
const REACTIONS_DB_NAME = 'reactions_elia_nicolo'

// URLs complètes avec auth
const REMOTE_POSTS_URL = `http://${USERNAME}:${PASSWORD}@${COUCHDB_URL.replace('http://', '')}/${POSTS_DB_NAME}`
const REMOTE_REACTIONS_URL = `http://${USERNAME}:${PASSWORD}@${COUCHDB_URL.replace('http://', '')}/${REACTIONS_DB_NAME}`

// ==================== INITIALISATION ====================
const initDatabase = async () => {
  console.log('🔧 Initialisation des bases...')

  // Créer les bases locales
  postsDB.value = new PouchDB<Post>('local_posts')
  reactionsDB.value = new PouchDB<Reaction>('local_reactions')
  console.log('✅ Bases locales créées')

  // Créer les index
  try {
    await postsDB.value.createIndex({
      index: { fields: ['post_name'] }
    })
    await postsDB.value.createIndex({
      index: { fields: ['total_likes'] }
    })
    await reactionsDB.value.createIndex({
      index: { fields: ['post_id', 'user_id'] }
    })
    console.log('✅ Index créés')
  } catch (err) {
    console.error('❌ Erreur création index:', err)
  }

  // Vérifier la connexion CouchDB
  await checkCouchDBConnection()

  // Synchronisation initiale
  await performInitialSync()

  // Charger les données
  await fetchData()

  // Démarrer le live sync
  startLiveSync()
}

// ==================== VÉRIFICATION COUCHDB ====================
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

  // Vérifier les bases
  try {
    const authHeader = 'Basic ' + btoa(`${USERNAME}:${PASSWORD}`)
    const response = await fetch(`${COUCHDB_URL}/_all_dbs`, {
      headers: { 'Authorization': authHeader }
    })
    const dbs = await response.json()
    console.log('📦 Bases disponibles:', dbs)

    if (!dbs.includes(POSTS_DB_NAME)) {
      console.error(`❌ La base ${POSTS_DB_NAME} n'existe pas`)
      throw new Error(`Base ${POSTS_DB_NAME} manquante`)
    }
    if (!dbs.includes(REACTIONS_DB_NAME)) {
      console.error(`❌ La base ${REACTIONS_DB_NAME} n'existe pas`)
      throw new Error(`Base ${REACTIONS_DB_NAME} manquante`)
    }
    console.log('✅ Les deux bases existent')
  } catch (err) {
    console.error('❌ Erreur vérification bases:', err)
    throw err
  }
}

// ==================== SYNCHRONISATION INITIALE ====================
const performInitialSync = async () => {
  if (!postsDB.value || !reactionsDB.value) return

  console.log('🔄 Synchronisation initiale...')

  try {
    // Sync posts
    await postsDB.value.replicate.from(REMOTE_POSTS_URL)
    console.log('✅ Posts synchronisés depuis le serveur')

    // Sync reactions
    await reactionsDB.value.replicate.from(REMOTE_REACTIONS_URL)
    console.log('✅ Reactions synchronisées depuis le serveur')
  } catch (err) {
    console.error('❌ Erreur sync initiale:', err)
    throw err
  }
}

// ==================== SYNCHRONISATION LIVE ====================
const startLiveSync = () => {
  if (!postsDB.value || !reactionsDB.value) {
    console.log('⚠️ DBs non initialisées, impossible de démarrer la sync')
    return
  }

  if (isOffline.value) {
    console.log('⏸️ Mode offline : sync live non démarrée')
    return
  }

  console.log('🔄 Démarrage sync bidirectionnelle live...')

  // Sync POSTS bidirectionnel
  postsSyncHandler = postsDB.value.sync(REMOTE_POSTS_URL, {
    live: true,
    retry: true
  })
    .on('change', (info) => {
      console.log('📦 Changement posts:', info.direction, 'docs:', info.change.docs.length)
      fetchData()
    })
    .on('paused', (err) => {
      if (err) {
        console.error('❌ Sync posts en pause suite à erreur:', err)
      } else {
        console.log('⏸️ Sync posts en pause (à jour)')
      }
    })
    .on('active', () => {
      console.log('▶️ Sync posts reprise')
    })
    .on('error', (err) => {
      console.error('❌ Erreur sync posts:', err)
    })

  // Sync REACTIONS bidirectionnel
  reactionsSyncHandler = reactionsDB.value.sync(REMOTE_REACTIONS_URL, {
    live: true,
    retry: true
  })
    .on('change', (info) => {
      console.log('💬 Changement reactions:', info.direction, 'docs:', info.change.docs.length)
      fetchData()
    })
    .on('paused', (err) => {
      if (err) {
        console.error('❌ Sync reactions en pause suite à erreur:', err)
      } else {
        console.log('⏸️ Sync reactions en pause (à jour)')
      }
    })
    .on('active', () => {
      console.log('▶️ Sync reactions reprise')
    })
    .on('error', (err) => {
      console.error('❌ Erreur sync reactions:', err)
    })

  console.log('✅ Sync live activée')
}

const stopLiveSync = () => {
  if (postsSyncHandler) {
    postsSyncHandler.cancel()
    postsSyncHandler = null
    console.log('🛑 Sync posts arrêtée')
  }
  if (reactionsSyncHandler) {
    reactionsSyncHandler.cancel()
    reactionsSyncHandler = null
    console.log('🛑 Sync reactions arrêtée')
  }
}

// ==================== SYNC MANUELLE ====================
const manualSync = async () => {
  if (!postsDB.value || !reactionsDB.value) return

  // La sync manuelle n'a de sens qu'en mode offline
  if (!isOffline.value) {
    console.log('⚠️ En mode online, la sync est automatique')
    return
  }

  try {
    console.log('🔄 Synchronisation manuelle (mode offline)...')

    // Sync bidirectionnelle one-shot
    await postsDB.value.sync(REMOTE_POSTS_URL)
    await reactionsDB.value.sync(REMOTE_REACTIONS_URL)

    await fetchData()

    showSyncMessage.value = true
    setTimeout(() => {
      showSyncMessage.value = false
    }, 3000)

    console.log('✅ Synchronisation manuelle terminée')
  } catch (err) {
    console.error('❌ Erreur synchronisation manuelle:', err)
    alert('Erreur de synchronisation. Vérifiez la console.')
  }
}

// ==================== MODE OFFLINE ====================
const toggleOffline = () => {
  isOffline.value = !isOffline.value

  if (isOffline.value) {
    stopLiveSync()
    console.log('🔴 MODE OFFLINE activé')
    console.log('   → Sync live arrêtée')
    console.log('   → Utilisez "Synchroniser manuellement" pour envoyer vos changements')
  } else {
    startLiveSync()
    console.log('🟢 MODE ONLINE activé')
    console.log('   → Sync live démarrée')
    console.log('   → Vos changements sont propagés automatiquement')
  }
}

// ==================== RÉCUPÉRATION DONNÉES ====================
const fetchData = async () => {
  if (!postsDB.value || !reactionsDB.value) return
  try {
    // EN MODE ONLINE : synchroniser avant de lire pour avoir les données à jour
    if (!isOffline.value) {
      console.log('🔄 Sync automatique avant lecture (mode online)...')
      await postsDB.value.sync(REMOTE_POSTS_URL)
      await reactionsDB.value.sync(REMOTE_REACTIONS_URL)
      console.log('✅ Données synchronisées depuis le serveur')
    }

    // Récupérer les posts avec détection de conflits
    const result = await postsDB.value.allDocs({
      include_docs: true,
      conflicts: true
    })

    postsData.value = result.rows
      .filter((row) => row.doc && !row.id.startsWith('_design/'))
      .map((row) => row.doc as Post)
      .sort((a, b) => {
        const dateA = a.attributes?.creation_date || ''
        const dateB = b.attributes?.creation_date || ''
        return dateB.localeCompare(dateA)
      })

    // Log les conflits
    const conflicted = postsData.value.filter(p => p._conflicts && p._conflicts.length > 0)
    if (conflicted.length > 0) {
      console.log('⚠️ CONFLITS DÉTECTÉS:', conflicted.map(p => ({
        id: p._id,
        name: p.post_name,
        conflicts: p._conflicts
      })))
    }

    // Récupérer les reactions
    const reactionsResult = await reactionsDB.value.allDocs({
      include_docs: true
    })

    reactionsData.value = reactionsResult.rows
      .filter((row) => row.doc && !row.id.startsWith('_design/'))
      .map((row) => row.doc as Reaction)

    console.log(`📊 ${postsData.value.length} posts, ${reactionsData.value.length} reactions`)
  } catch (err) {
    console.error('❌ Erreur fetchData:', err)
  }
}

// ==================== GESTION CONFLITS ====================
const resolveConflict = async (postId: string) => {
  if (!postsDB.value) return

  try {
    const post = await postsDB.value.get(postId, { conflicts: true })

    if (!post._conflicts || post._conflicts.length === 0) {
      console.log('Aucun conflit trouvé')
      return
    }

    selectedConflict.value = post
    otherVersions.value = []

    for (const rev of post._conflicts) {
      const conflictVersion = await postsDB.value.get(postId, { rev })
      otherVersions.value.push(conflictVersion)
    }

    console.log(`⚠️ ${post._conflicts.length} conflit(s) détecté(s)`)
  } catch (err) {
    console.error('❌ Erreur résolution conflit:', err)
  }
}

const keepLocal = async () => {
  if (!selectedConflict.value || !selectedConflict.value._conflicts || !postsDB.value) return

  try {
    // Supprimer toutes les versions conflictuelles
    for (const rev of selectedConflict.value._conflicts) {
      await postsDB.value.remove(selectedConflict.value._id, rev)
    }

    await fetchData()
    cancelConflictResolution()
    console.log('✅ Version locale conservée')
  } catch (err) {
    console.error('❌ Erreur keepLocal:', err)
  }
}

const keepRemote = async (index: number) => {
  if (!selectedConflict.value || !selectedConflict.value._conflicts || !postsDB.value) return

  try {
    const chosenVersion = otherVersions.value[index]

    // Mettre à jour avec la version choisie
    const newDoc: Post = {
      _id: selectedConflict.value._id,
      _rev: selectedConflict.value._rev,
      post_name: chosenVersion.post_name,
      post_content: chosenVersion.post_content,
      total_likes: chosenVersion.total_likes,
      attributes: chosenVersion.attributes
    }

    await postsDB.value.put(newDoc)

    // Supprimer les autres versions conflictuelles
    for (const rev of selectedConflict.value._conflicts) {
      await postsDB.value.remove(selectedConflict.value._id, rev)
    }

    await fetchData()
    cancelConflictResolution()
    console.log('✅ Version distante conservée')
  } catch (err) {
    console.error('❌ Erreur keepRemote:', err)
  }
}

const cancelConflictResolution = () => {
  selectedConflict.value = null
  otherVersions.value = []
}

// ==================== RECHERCHE ====================
const searchPosts = async (term: string) => {
  if (!postsDB.value) return

  if (!term.trim()) {
    await fetchData()
    return
  }

  try {
    const result = await postsDB.value.allDocs({ include_docs: true, conflicts: true })
    postsData.value = result.rows
      .map(row => row.doc!)
      .filter(doc =>
        !doc._id.startsWith('_design') &&
        doc.post_name.toLowerCase().includes(term.toLowerCase())
      )
  } catch (err) {
    console.error('❌ Erreur recherche:', err)
  }
}

const getTopLikedPosts = async () => {
  if (!postsDB.value) return

  try {
    const result = await postsDB.value.find({
      selector: {
        total_likes: { $gte: 0 }
      },
      sort: [{ total_likes: 'desc' }],
      limit: 10
    })
    postsData.value = result.docs
  } catch (err) {
    console.error('❌ Erreur top likes:', err)
  }
}

// ==================== CRUD POSTS ====================
const addDocument = async () => {
  if (!postsDB.value || !documentName.value.trim()) return

  try {
    const newPost: Post = {
      _id: `post_${Date.now()}`,
      post_name: documentName.value,
      post_content: documentContent.value || 'Contenu vide',
      total_likes: 0,
      attributes: {
        creation_date: new Date().toISOString()
      }
    }

    await postsDB.value.put(newPost)

    documentName.value = ''
    documentContent.value = ''
    await fetchData()
    console.log('✅ Post créé (sync auto via live sync)')
  } catch (err) {
    console.error('❌ Erreur ajout document:', err)
  }
}

const updateDocument = async (post: Post, newName: string) => {
  if (!postsDB.value || !post._rev || !newName.trim()) return

  try {
    await postsDB.value.put({
      _id: post._id,
      _rev: post._rev,
      post_name: newName,
      post_content: post.post_content,
      total_likes: post.total_likes,
      attributes: post.attributes
    })

    await fetchData()
    console.log('✅ Post modifié (sync auto via live sync)')
  } catch (err) {
    console.error('❌ Erreur update document:', err)
  }
}

const deleteDocument = async (id: string, rev?: string) => {
  if (!postsDB.value || !rev) return

  try {
    await postsDB.value.remove(id, rev)
    await fetchData()
    console.log('✅ Post supprimé (sync auto via live sync)')
  } catch (err) {
    console.error('❌ Erreur suppression document:', err)
  }
}

// ==================== REACTIONS ====================
const getReactionForPost = (postId: string): Reaction | null => {
  return reactionsData.value.find(
    r => r.post_id === postId && r.user_id === 'user_1'
  ) || null
}

const addReaction = async (postId: string, comment?: string, toggleLike?: boolean) => {
  if (!reactionsDB.value || !postsDB.value) return

  const reactionId = `reaction_${postId}_user_1`

  try {
    let reaction: Reaction | null = null

    try {
      reaction = await reactionsDB.value.get(reactionId)
    } catch (err: any) {
      if (err.status !== 404) throw err
    }

    if (reaction) {
      let updated = false

      if (toggleLike !== undefined) {
        reaction.isliked = toggleLike
        updated = true
      }

      if (comment && comment.trim()) {
        reaction.comments.push(comment.trim())
        updated = true
      }

      if (updated) {
        await reactionsDB.value.put(reaction)
      }
    } else {
      const newReaction: Reaction = {
        _id: reactionId,
        user_id: 'user_1',
        post_id: postId,
        isliked: toggleLike || false,
        comments: comment && comment.trim() ? [comment.trim()] : []
      }
      await reactionsDB.value.put(newReaction)
    }

    await updateTotalLikes(postId)
    newComment.value = ''
    await fetchData()
    console.log('✅ Réaction ajoutée (sync auto via live sync)')
  } catch (err) {
    console.error('❌ Erreur ajout réaction:', err)
  }
}

const updateTotalLikes = async (postId: string) => {
  if (!postsDB.value || !reactionsDB.value) return

  try {
    const result = await reactionsDB.value.find({
      selector: {
        post_id: postId,
        isliked: true
      }
    })

    const likeCount = result.docs.length

    const post = await postsDB.value.get(postId)
    post.total_likes = likeCount
    await postsDB.value.put(post)

    console.log(`✅ Post ${postId} : ${likeCount} likes`)
  } catch (err) {
    console.error('❌ Erreur mise à jour total_likes:', err)
  }
}

const deleteComment = async (postId: string, commentText: string) => {
  if (!reactionsDB.value) return

  const reactionId = `reaction_${postId}_user_1`

  try {
    const reaction = await reactionsDB.value.get(reactionId)
    reaction.comments = reaction.comments.filter(c => c !== commentText)
    await reactionsDB.value.put(reaction)
    await fetchData()
    console.log('✅ Commentaire supprimé (sync auto via live sync)')
  } catch (err) {
    console.error('❌ Erreur suppression commentaire:', err)
  }
}

// ==================== LIFECYCLE ====================
onMounted(async () => {
  try {
    await initDatabase()
  } catch (err) {
    console.error('❌ Erreur initialisation:', err)
    alert('Erreur d\'initialisation. Vérifiez que CouchDB est démarré et accessible.')
  }
})

onUnmounted(() => {
  stopLiveSync()
})
</script>
<template>
  <div style="padding: 20px; max-width: 800px; margin: 0 auto;">
    <h1>Gestion Posts & Réactions</h1>

    <div style="background: #f5f5f5; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
      <button @click="manualSync" :disabled="!isOffline" :style="{ opacity: isOffline ? 1 : 0.5 }">
        🔄 Synchroniser manuellement
      </button>

      <button @click="toggleOffline" :style="{ background: isOffline ? '#f44336' : '#4caf50', color: 'white' }">
        {{ isOffline ? '🔴 Hors ligne' : '🟢 En ligne' }}
      </button>
      <button @click="fetchData">🔥 Rafraîchir</button>
      <button @click="getTopLikedPosts">⭐ Top likes</button>

      <div v-if="showSyncMessage" style="color: green; margin-top: 10px;">
        ✅ Synchronisation réussie
      </div>

      <div v-if="isOffline"
        style="background: #fff3cd; color: #856404; padding: 10px; margin-top: 10px; border-radius: 4px; font-weight: bold;">
        🔴 MODE HORS LIGNE<br>
        <span style="font-weight: normal; font-size: 0.9em;">
          ⚠️ Sync automatique désactivée. Cliquez sur "Synchroniser manuellement" pour envoyer vos modifications.
        </span>
      </div>

      <div v-if="!isOffline"
        style="background: #d4edda; color: #155724; padding: 10px; margin-top: 10px; border-radius: 4px; font-weight: bold;">
        🟢 MODE EN LIGNE<br>
        <span style="font-weight: normal; font-size: 0.9em;">
          ✅ Synchronisation automatique active. Vos modifications sont envoyées en temps réel.
        </span>
      </div>

      <div style="margin-bottom: 20px;">
        <input v-model="searchTerm" @input="searchPosts(searchTerm)" placeholder="🔍 Rechercher un post..."
          style="width: 100%; padding: 10px; font-size: 16px;" />
      </div>

      <div style="background: #e3f2fd; padding: 15px; margin-bottom: 20px; border-radius: 8px;">
        <h3>➕ Nouveau post</h3>
        <input v-model="documentName" placeholder="Nom du post"
          style="width: 100%; margin-bottom: 10px; padding: 8px;" />
        <textarea v-model="documentContent" placeholder="Contenu du post" rows="3"
          style="width: 100%; margin-bottom: 10px; padding: 8px;"></textarea>
        <button @click="addDocument" :disabled="!documentName.trim()">Créer le post</button>
      </div>

      <div v-if="selectedConflict" style="background: #fff3cd; padding: 20px; margin-bottom: 20px; border-radius: 8px;">
        <h2>⚠️ Résolution de conflit</h2>

        <div style="background: #d4edda; padding: 10px; margin: 10px 0; border-radius: 4px;">
          <h4>Version locale (actuelle)</h4>
          <p><strong>Nom:</strong> {{ selectedConflict.post_name }}</p>
          <p><strong>Contenu:</strong> {{ selectedConflict.post_content }}</p>
          <p><strong>Likes:</strong> {{ selectedConflict.total_likes }}</p>
          <button @click="keepLocal" style="background: #28a745; color: white;">Garder cette version</button>
        </div>

        <div v-for="(version, idx) in otherVersions" :key="idx"
          style="background: #fff0f0; padding: 10px; margin: 10px 0; border-radius: 4px;">
          <h4>Version distante {{ idx + 1 }}</h4>
          <p><strong>Nom:</strong> {{ version.post_name }}</p>
          <p><strong>Contenu:</strong> {{ version.post_content }}</p>
          <p><strong>Likes:</strong> {{ version.total_likes }}</p>
          <button @click="keepRemote(idx)" style="background: #dc3545; color: white;">Garder cette version</button>
        </div>

        <button @click="cancelConflictResolution"
          style="margin-top: 10px; background: #6c757d; color: white;">Annuler</button>
      </div>

      <div v-if="!selectedConflict">
        <h2>📄 Posts ({{ postsData.length }})</h2>

        <article v-for="post in postsData" :key="post._id"
          style="border: 1px solid #ccc; padding: 15px; margin: 15px 0; border-radius: 8px;">
          <div v-if="post._conflicts && post._conflicts.length > 0"
            style="background: #ffebee; padding: 10px; margin-bottom: 15px; border-radius: 4px;">
            <span style="color:red; font-weight: bold;">
              ⚠️ Conflit détecté ({{ post._conflicts.length }} version(s))
            </span>
            <button @click="resolveConflict(post._id)" style="margin-left: 10px; background: #ff9800; color: white;">
              Résoudre le conflit
            </button>
          </div>

          <h3>{{ post.post_name }}</h3>
          <p>{{ post.post_content }}</p>
          <p style="font-size: 0.85em; color: #666;">
            💾 ID: {{ post._id }} | 👍 {{ post.total_likes || 0 }} likes
          </p>

          <div v-if="getReactionForPost(post._id)"
            style="background: #f9f9f9; padding: 10px; margin: 10px 0; border-radius: 4px;">
            <p v-if="getReactionForPost(post._id)!.isliked"
              style="margin: 0 0 10px 0; color: #4caf50; font-weight: bold;">
              ✅ Vous aimez ce post
            </p>
            <p v-else style="margin: 0 0 10px 0; color: #999;">
              Vous n'avez pas liké
            </p>

            <div v-if="getReactionForPost(post._id)!.comments.length > 0">
              <p style="margin: 10px 0 5px 0;">
                <strong>💬 Commentaires ({{ getReactionForPost(post._id)!.comments.length }}) :</strong>
              </p>
              <ul style="list-style: none; padding-left: 0; margin: 0;">
                <li v-for="(c, idx) in getReactionForPost(post._id)!.comments" :key="idx"
                  style="margin: 5px 0; padding: 8px; background: #fff; border: 1px solid #ddd; border-radius: 4px; display: flex; justify-content: space-between; align-items: center;">
                  <span>{{ c }}</span>
                  <button @click="deleteComment(post._id, c)"
                    style="background: #f44336; color: white; padding: 4px 8px; font-size: 0.85em; border: none; border-radius: 3px; cursor: pointer;">
                    🗑️
                  </button>
                </li>
              </ul>
            </div>
          </div>

          <div style="background: #f0f0f0; padding: 10px; margin: 10px 0; border-radius: 4px;">
            <button @click="addReaction(post._id, undefined, true)">👍 Like</button>
            <button @click="addReaction(post._id, undefined, false)">👎 Unlike</button>
            <input v-model="newComment" placeholder="Ajouter un commentaire" style="width: 60%; padding: 8px;" />
            <button @click="addReaction(post._id, newComment)" :disabled="!newComment.trim()">💬 Commenter</button>
          </div>

          <div style="margin-top: 15px; padding-top: 10px; border-top: 1px solid #ddd;">
            <input :value="post.post_name" @change="(e) => updateDocument(post, (e.target as HTMLInputElement).value)"
              placeholder="Nouveau nom" style="width: 60%; padding: 8px;" />
            <button @click="deleteDocument(post._id, post._rev)" style="background: #f44336; color: white;">
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
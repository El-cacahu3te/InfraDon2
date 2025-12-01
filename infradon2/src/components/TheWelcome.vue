<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

interface Post {
  _id: string
  _rev?: string
  post_name: string
  post_content: string
  total_likes?: number //counter 
  comments?: string[]
  attributes?: {
    creation_date: any
  }
  _conflicts?: string[]
}


interface Reaction {
  _id: string;           //  reaction_postId
  _rev?: string;         // révision
  user_id: string;
  post_id: string;       // lien vers le post
  isliked: boolean;
  comments: string[];
}


//Variables réactives
const documentName = ref('')
const documentNewName = ref<string[]>([])
const documentContent = ref('')
const storage = ref()
const postsData = ref<Post[]>([])
const showSyncMessage = ref(false)
const isOffline = ref(false)
const selectedConflict = ref<any>(null)
const otherVersions = ref<any[]>([])
const buttonName = ref("En ligne")
const searchTerm = ref('');
const newComment = ref('');
const reactionsData = ref<Reaction[]>([]);

let syncHandler: any = null

//Initialisation db et réplication initiale
const initDatabase = () => {
  console.log('=> Connexion à la base locale')
  const db = new PouchDB('collection_infradon2')
  storage.value = db

  // Réplication initiale depuis CouchDB
  db.replicate
    .from('http://elia:admin@localhost:5984/infradon2')
    .then(() => fetchData())
    .catch(err => console.error('Erreur réplication initiale:', err))

  storage.value.createIndex({
    index: { fields: ['post_name'] } //index pour la recherche par nom de post
  }).then(() => {
    console.log('Index créé sur post_name');
  });

  storage.value.createIndex({ //index pour chaque post et user
    index: { fields: ['post_id', 'user_id'] }
  }).then(() => {
    console.log('Index créé sur post_id et user_id');
  });

  storage.value.createIndex({ //index pour chaque post pour les réactions
    index: { fields: ['post_id'] }
  }).then(() => {
    console.log('Index créé sur post_id');
  });

  storage.value.createIndex({ //index pour trier par total_likes
    index: { fields: ['total_likes'] }
  }).then(() => {
    console.log('Index créé sur total_likes');
  });
}

// Rafraîchir (réplication ponctuelle)
const syncDatabase = () => {
  PouchDB.sync(storage.value, 'http://elia:admin@localhost:5984/infradon2')
    .then(() => {
      fetchData()
      showSyncMessage.value = true
      setTimeout(() => (showSyncMessage.value = false), 3000)
    })
    .catch(err => console.error('Erreur réplication:', err))
}

// Toggle Online/Offline
const toggleOffline = () => {
  isOffline.value = !isOffline.value
  buttonName.value = isOffline.value ? 'Hors ligne' : 'En ligne'

  if (isOffline.value) {
    // Stop live sync
    if (syncHandler) {
      syncHandler.cancel()
      console.log('Mode OFFLINE activé')
    }
  } else {
    // Restart live sync SANS résolution automatique
    syncHandler = storage.value.sync('http://elia:admin@localhost:5984/infradon2', {
      live: true,
      retry: true
    })
      .on('change', (info) => {
        console.log('Changement détecté:', info);
        fetchData(); // Rafraîchit pour voir les conflits
      })
      .on('error', (err) => {
        console.error('Erreur sync:', err);
      });

    console.log('Mode ONLINE activé')
  }
}

// Récupérer les données
const fetchData = () => {
  storage.value
    .allDocs({ include_docs: true, conflicts: true })
    .then((result: any) => {
      const allDocs = result.rows
        .map((row: any) => row.doc)
        .filter(doc => !doc._id.startsWith('_'))

      postsData.value = allDocs.filter(doc => !doc._id.startsWith('reaction_')) //données post
      reactionsData.value = allDocs.filter(doc => doc._id.startsWith('reaction_')) //données réactions

      console.log('Posts:', postsData.value.length);
      console.log('Réactions:', reactionsData.value.length);
      console.log('Documents avec conflits:', postsData.value.filter(p => p._conflicts));
      documentNewName.value = postsData.value.map(post => post.post_name)
    })
    .catch(err => console.error('Erreur récupération:', err))
};

// Résoudre conflit 
const resolveConflict = async (id: string) => {
  const doc = await storage.value.get(id, { conflicts: true });
  console.log('Version active:', doc);
  console.log('Révisions en conflit:', doc._conflicts);

  selectedConflict.value = doc;
  otherVersions.value = [];

  for (const rev of doc._conflicts) {
    const otherDoc = await storage.value.get(id, { rev });
    otherVersions.value.push(otherDoc);
  }
};

const keepLocal = async () => {
  // Supprimer toutes les versions en conflit
  for (const rev of selectedConflict.value._conflicts) {
    await storage.value.remove(selectedConflict.value._id, rev);
  }
  fetchData();
  selectedConflict.value = null;
  otherVersions.value = [];
};

const keepRemote = async (index: number) => {
  const chosenVersion = otherVersions.value[index];

  // Mettre à jour avec la version distante qui est recréer en local 
  const newDoc = {
    ...selectedConflict.value, //sturcture local
    post_name: chosenVersion.post_name,//contenu distant
    post_content: chosenVersion.post_content,
    total_likes: chosenVersion.total_likes,
    comments: chosenVersion.comments,
  };
  await storage.value.put(newDoc);
  for (const rev of selectedConflict.value._conflicts) {
    await storage.value.remove(selectedConflict.value._id, rev);
  } //supprimer les autres versions en conflit

  fetchData();
  selectedConflict.value = null;
  otherVersions.value = [];
};

// Recherche de posts par nom
const searchPosts = async (term: string) => {
  if (!term || term.trim() === '') {
    fetchData();  // Si vide, recharger tout
    return;
  }

  try {
    // l'index sur post_name
    const result = await storage.value.find({
      selector: {
        post_name: {
          $regex: new RegExp(term, 'i')  // 'i' = case insensitive
        }
      }
    });

    postsData.value = result.docs.filter(doc => !doc._id.startsWith('reaction_'));
  } catch (err) {
    console.error('Erreur recherche:', err);
  }
};

// Récupérer le top 10 des posts les plus likés
const getTopLikedPosts = async () => {
  try {
    // Utilise l'index sur total_likes
    const result = await storage.value.find({
      selector: {
        total_likes: { $gte: 0 }  // Tous les posts avec des likes >= 0
      },
      sort: [{ total_likes: 'desc' }],
      limit: 10
    });

    postsData.value = result.docs.filter(doc => !doc._id.startsWith('reaction_'));
    console.log('Top 10 posts avec le plus de likes:', postsData.value);
  } catch (err) {
    console.error('Erreur récupération top likes:', err);
  }
};


const addDocument = () => {
  storage.value
    .post({
      post_name: documentName.value,
      post_content: 'blabla',
      total_likes: 0,
      comments: []
    })
    .then(() => {
      documentName.value = '';
      fetchData();
    })
    .catch(err => console.log(err))
}


const updateDocument = (post_id: string, post_rev: string, index: number) => {
  storage.value
    .put({
      _id: post_id,
      _rev: post_rev,
      post_name: documentNewName.value[index],
      post_content: documentContent.value + ' - updated ' + new Date().toISOString()
    })
    .then(() => fetchData())
    .catch(err => console.log(err))
}


const deleteDocument = (post_id: string, post_rev: string) => {
  storage.value
    .remove(post_id, post_rev)
    .then(() => fetchData())
    .catch(err => console.log(err))
}

// Factory pour générer 50 documents aléatoires
const generateFactory = () => {
  const docs = []
  for (let i = 0; i < 50; i++) {
    docs.push({
      _id: 'doc_' + i + '_' + Date.now(),
      post_name: 'Post ' + i,
      post_content: 'Contenu aléatoire ' + Math.random().toString(36).substring(7),
      total_likes: Math.floor(Math.random() * 100),
      comments: []
    })
  }
  storage.value.bulkDocs(docs).then(() => fetchData())
}

const cancelConflictResolution = () => {
  selectedConflict.value = null;
  otherVersions.value = [];
}

const addReaction = async (post_id: string, comment?: string, isliked?: boolean) => {
  try { //indexer pour chaque post et user
    const existing = await storage.value.find({
      selector: {
        post_id: post_id,
        user_id: 'user_1'
      }
    });

    if (existing.docs.length > 0) {
      // Modifier la réaction existante
      const reaction = existing.docs[0];
      if (comment && comment.trim()) {  // conserver uniquement les commentaires non vides
        reaction.comments.push(comment);
      }
      if (isliked !== undefined) {
        reaction.isliked = isliked;
      }
      await storage.value.put(reaction);
      await cleanReaction(post_id); //nettoyage si vide
    } else { // créer une nouvelle réaction    
      const newReaction: Reaction = {
        _id: 'reaction_' + post_id + '_user1',  //ID unique
        post_id,
        user_id: 'user_1',
        isliked: isliked || false,
        comments: comment && comment.trim() ? [comment] : []
      };
      await storage.value.put(newReaction);
    }

    newComment.value = '';   // reset le champ commentaire
    fetchData();
  } catch (err) {
    console.error('Erreur ajout réaction:', err);
  }
};


const cleanReaction = async (post_id: string) => {
  const existing = await storage.value.find({
    selector: { post_id: post_id, user_id: 'user_1' }
  });
  if (existing.docs.length > 0) {
    const reaction = existing.docs[0];
    if (!reaction.isliked && reaction.comments.length === 0) {
      await storage.value.remove(reaction._id, reaction._rev);
    }
  }
};


const getReactionForPost = (post_id: string) => {
  return reactionsData.value.find(r => r.post_id === post_id && r.user_id === 'user_1') || null;
};

// Supprimer un commentaire spécifique
const deleteComment = async (post_id: string, comment: string) => {
  try {
    const reaction = getReactionForPost(post_id);
    if (reaction && reaction._rev) { // Assure que la réaction existe
      const updatedReaction = {
        ...reaction,
        comments: reaction.comments.filter(c => c !== comment)
      };
      await storage.value.put(updatedReaction);
      await cleanReaction(post_id);
      fetchData();
    }
  } catch (err) {
    console.error('Erreur suppression commentaire:', err);
  }
  
};

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
}); 
</script>

<template>
  <h1>Gestion des données</h1>

  <!-- Boutons de contrôle -->
  <div style="margin-bottom: 20px;">
    <button @click="syncDatabase">Rafraîchir les données</button>
    <button @click="toggleOffline">Vous êtes: {{ buttonName }}</button>
    <br></br>
    <input v-model="documentName" placeholder="Nom du document" />
    <button @click="addDocument">Ajouter un document</button>
    <br></br>
    <input v-model="searchTerm" placeholder="Rechercher un post" />
    <button @click="searchPosts(searchTerm)">Rechercher</button>
    <br></br>
    <button @click="generateFactory">Générer 50 documents</button>
    <button @click="getTopLikedPosts">Top 10 Likes</button>
  </div>

  <p v-if="showSyncMessage" style="color: green; font-weight: bold;">
    ✓ Base synchronisée
  </p>

  <!-- Zone de résolution des conflits -->
  <div v-if="selectedConflict" style="border: 2px solid red; padding: 20px; margin: 20px 0;">
    <h3>⚠️ Résolution du conflit pour {{ selectedConflict._id }}</h3>

    <div style="background: #f0f0f0; padding: 10px; margin: 10px 0;">
      <h4>Version locale (actuelle)</h4>
      <p><strong>Nom:</strong> {{ selectedConflict.post_name }}</p>
      <p><strong>Contenu:</strong> {{ selectedConflict.post_content }}</p>
      <button @click="keepLocal()">Garder la version locale</button>
    </div>

    <div v-for="(version, idx) in otherVersions" :key="idx" style="background: #fff0f0; padding: 10px; margin: 10px 0;">
      <h4>Version distante {{ idx + 1 }}</h4>
      <p><strong>Nom:</strong> {{ version.post_name }}</p>
      <p><strong>Contenu:</strong> {{ version.post_content }}</p>
      <button @click="keepRemote(idx)">Garder cette version</button>
    </div>

    <button @click="cancelConflictResolution" style="margin-top: 10px;">Annuler</button>
  </div>

  <!-- Liste des documents -->
  <div v-if="!selectedConflict">
    <h2>Documents ({{ postsData.length }})</h2>

    <article v-for="(post, index) in postsData" :key="post._id"
      style="border: 1px solid #ccc; padding: 15px; margin: 15px 0; border-radius: 8px;">

      <!-- Alerte conflit (en haut) -->
      <div v-if="post._conflicts" style="background: #ffebee; padding: 10px; margin-bottom: 15px; border-radius: 4px;">
        <span style="color:red; font-weight: bold;">⚠️ Conflit détecté ({{ post._conflicts.length }} version(s))</span>
        <button @click="resolveConflict(post._id)" style="margin-left: 10px;">Voir les versions</button>
      </div>

      <!-- Contenu du post -->
      <h3>{{ post.post_name }}</h3>
      <p>{{ post.post_content }}</p>
      <p style="font-size: 0.8em; color: #666;">ID: {{ post._id }}</p>
      <!-- Affichage des réactions (likes) -->
      <div v-if="getReactionForPost(post._id)"
        style="background: #f9f9f9; padding: 10px; margin: 10px 0; border-radius: 4px;">
        <p style="margin: 0 0 10px 0;">👍 Likes</p>
        <button @click="addReaction(post._id, undefined, false)">👎 Unlike</button>

        <!-- Liste complète des commentaires avec bouton supprimer -->
        <div v-if="getReactionForPost(post._id).comments.length > 0">
          <p style="margin: 10px 0 5px 0;"><strong>💬 Commentaires ({{ getReactionForPost(post._id).comments.length }})
              :</strong></p>
          <ul style="list-style: none; padding-left: 0; margin: 0;">
            <li v-for="(c, idx) in getReactionForPost(post._id).comments" :key="idx"
              style="margin: 5px 0; padding: 8px; background: #fff; border: 1px solid #ddd; border-radius: 4px; display: flex; justify-content: space-between; align-items: center;">
              <span>{{ c }}</span>
              <button @click="deleteComment(post._id, c)"
                style="background: #f44336; color: white; padding: 4px 8px; font-size: 0.85em; border: none; border-radius: 3px; cursor: pointer;">
                🗑️ Supprimer
              </button>
            </li>
          </ul>
        </div>
      </div>

      <!-- Section actions pour ajouter des réactions -->
      <div style="background: #f0f0f0; padding: 10px; margin: 10px 0; border-radius: 4px;">
        <button @click="addReaction(post._id, undefined, true)">👍 Like</button>
        <input v-model="newComment" placeholder="Ajouter un commentaire" />
        <button @click="addReaction(post._id, newComment)">💬 Commenter</button>
      </div>

      <!-- Actions sur le document -->
      <div style="margin-top: 15px; padding-top: 10px; border-top: 1px solid #ddd;">
        <input v-model="documentNewName[index]" placeholder="Nouveau nom" />
        <button @click="updateDocument(post._id, post._rev, index)">✏️ Modifier</button>
        <button @click="deleteDocument(post._id, post._rev)" style="background: #f44336; color: white;">🗑️
          Supprimer</button>
      </div>
    </article>
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

input {
  margin: 5px;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

article {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
</style>
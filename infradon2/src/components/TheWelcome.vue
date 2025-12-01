<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);


// on dirait que le declare ne sert a rien
declare interface Post {
  _id: string
  _rev: string
  post_name: string
  post_content: string
  likes?: number
  comments?: string[]
  attributes?: {
    creation_date: any
  }
  _conflicts?: string[]
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
const buttonName = ref(window.navigator.onLine ? 'En ligne' : 'Hors ligne')

let syncHandler: any = null // Pour gérer la synchronisation live

//Initialisation de la base locale et réplication initiale
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
  index: { fields: ['post_name'] }
}).then(() => {
  console.log('Index créé sur post_name');
});

}

// Rafraîchir (réplication ponctuelle)
const syncDatabase = () => {
  PouchDB.sync(storage.value, 'http://elia:admin@localhost:5984/infradon2') //récupère la db en ligne juste 1 fois 
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
    // Restart live sync
    syncHandler = PouchDB.sync(storage.value, 'http://elia:admin@localhost:5984/infradon2', {
      live: true,
      retry: true
    }).on('change', fetchData)
    console.log('Mode ONLINE activé')
  }
}

// Récupérer les données
const fetchData = () => {
  storage.value
    .allDocs({ include_docs: true, conflicts: true }) //include_docs pour obtenir les documents complets + annonce les conflits 
    .then((result: any) => {
      postsData.value = result.rows
        .map((row: any) => row.doc) //done un tableau des documents complets
        .filter(doc => !doc._id.startsWith('_')) // Exclure les documents systèmes
      console.log(postsData.value); // regarde si certains ont des conflits 

      documentNewName.value = postsData.value.map(post => post.post_name)
    })

    .catch(err => console.error('Erreur récupération:',))
}

//résoudre conflit 
const resolveConflict = async (id: string) => {
  // Récupère le document avec ses conflits
  const doc = await storage.value.get(id, { conflicts: true, revs: true });
  console.log('Version active:', doc);
  console.log('Révisions en conflit:', doc._conflicts); //tableau de conflits 
  selectedConflict.value = doc; //stock document actif
otherVersions.value = [];
for (const rev of doc._conflicts) {
  const otherDoc = await storage.value.get(id, { rev });
  otherVersions.value.push(otherDoc);
} 
selectedConflict.value = null;
otherVersions.value = [];

};


const keepLocal = async () => {
  for (const rev of selectedConflict.value._conflicts) {
    await storage.value.remove(selectedConflict.value._id, rev);
  }
  fetchData();
  
selectedConflict.value = null;
otherVersions.value = [];

};

const keepRemote = async (index: number) => {
const idx = index;
await storage.value.put(otherVersions.value[idx]);
for (const rev of selectedConflict.value._conflicts) {
  if (rev !== otherVersions.value[idx]._rev) {
    await storage.value.remove(selectedConflict.value._id, rev);
  }
}
fetchData();

selectedConflict.value = null;
otherVersions.value = [];

};

const mergeVersions = async () => { 
const idx = 0; // Exemple : fusionner avec la première version distante
const mergedContent = selectedConflict.value.post_content + ' | ' + otherVersions.value[idx].post_content;
await storage.value.put({ ...selectedConflict.value, post_content: mergedContent });
for (const rev of selectedConflict.value._conflicts) {
  await storage.value.remove(selectedConflict.value._id, rev);
}
fetchData();

};


// Ajouter un document
const addDocument = () => {
  storage.value
    .post({
      post_name: documentName.value,
      post_content: 'blabla',
      likes: 0,
      comments: []
    })
    .then(() => fetchData())
    .catch(err => console.log(err))
}

// Modifier un document
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

//Supprimer un document
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
      likes: Math.floor(Math.random() * 100),
      comments: []
    })
  }
  storage.value.bulkDocs(docs).then(() => fetchData())
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})
</script>

<template>
  <h1>Gestion des données</h1>

  <!-- Boutons -->
  <button @click="syncDatabase">Rafraîchir les données</button>
  <button @click="toggleOffline">{{ buttonName }}</button>
  <p v-if="showSyncMessage" style="color: green; font-weight: bold;">
    ✓ Base synchronisée
  </p>
  
<article v-for="(post, index) in postsData" :key="post._id">
  <h2>{{ post.post_name }}</h2>
  <p>{{ post.post_content }}</p>

<!-- Zone d'affichage des versions en conflit -->
<div v-if="selectedConflict">
  <h3>Résolution du conflit pour {{ selectedConflict._id }}</h3>
  <br></br>

  <!-- Ajout document -->
  <button @click="addDocument">Ajouter un document</button>
  <input v-model="documentName" placeholder="Nom du document" />
  <br></br>
  <button @click="generateFactory">Générer 50 documents</button>
 //gestion des conflits
   <div>
    <h4>Version locale</h4>
    <p>{{ selectedConflict.post_content }}</p>
    <button @click="keepLocal()">Garder locale</button>
  </div>

  <!-- Autres versions -->
  <div v-for="(version, idx) in otherVersions" :key="idx">
    <h4>Version distante {{ idx + 1 }}</h4>
    <p>{{ version.post_content }}</p>
    <button @click="keepRemote(idx)">Garder cette version</button>
  </div>

  
 <div>
    <button @click="mergeVersions()">Fusionner</button>
  </div>
</div>
  <!-- Liste des documents -->
  <article v-for="(post, index) in postsData" :key="post._id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
    <p>ID: {{ post._id }}</p>
    <input v-model="documentNewName[index]" placeholder="Nouveau nom" />
    <button @click="updateDocument(post._id, post._rev, index)">Modifier</button>
    <button @click="deleteDocument(post._id, post._rev)">Supprimer</button>
  </article>
</template>

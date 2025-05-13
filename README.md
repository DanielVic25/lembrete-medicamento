# lembrete-medicamento

```
import React, { useState, useEffect } from 'react';
import {
  View, Text, TextInput, FlatList, Alert, TouchableOpacity, Image,
  StyleSheet, KeyboardAvoidingView, Platform, ScrollView, Modal
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

// ----------- TELA DE LOGIN -----------
function LoginScreen({ navigation }) {
  const [username, setUsername] = useState('');

  const handleLogin = () => {
    if (username.trim() === '') {
      Alert.alert('Erro', 'Digite seu nome.');
      return;
    }

    navigation.replace('Início');
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>🔐 Login</Text>
      <TextInput
        placeholder="Digite seu nome"
        value={username}
        onChangeText={setUsername}
        style={styles.input}
      />
      <TouchableOpacity style={styles.addButton} onPress={handleLogin}>
        <Text style={styles.addButtonText}>Entrar</Text>
      </TouchableOpacity>
    </View>
  );
}

// ----------- TELA PRINCIPAL -----------
function HomeScreen({ navigation }) {
  const [medicationName, setMedicationName] = useState('');
  const [medicationTime, setMedicationTime] = useState('');
  const [medications, setMedications] = useState([]);
  const [isModalVisible, setIsModalVisible] = useState(false);
  const [selectedMedication, setSelectedMedication] = useState(null);
  const [isEditing, setIsEditing] = useState(false);

  useEffect(() => { loadMedications(); }, []);
  useEffect(() => { saveMedications(); }, [medications]);

  const saveMedications = async () => {
    try { await AsyncStorage.setItem('medications', JSON.stringify(medications)); }
    catch (e) { console.error('Erro ao salvar:', e); }
  };

  const loadMedications = async () => {
    try {
      const stored = await AsyncStorage.getItem('medications');
      if (stored) setMedications(JSON.parse(stored));
    } catch (e) { console.error('Erro ao carregar:', e); }
  };

  const openModal = (medication) => {
    setSelectedMedication(medication);
    setMedicationName(medication.name);
    setMedicationTime(medication.time);
    setIsEditing(true);
    setIsModalVisible(true);
  };

  const closeModal = () => {
    setIsModalVisible(false);
    setMedicationName('');
    setMedicationTime('');
    setSelectedMedication(null);
    setIsEditing(false);
  };

  const addOrUpdateMedication = () => {
    if (!medicationName || !medicationTime) {
      Alert.alert('Erro', 'Preencha todos os campos.');
      return;
    }

    if (isEditing && selectedMedication) {
      setMedications(medications.map(med =>
        med.id === selectedMedication.id
          ? { ...med, name: medicationName, time: medicationTime }
          : med
      ));
    } else {
      const newMedication = {
        id: Date.now().toString(),
        name: medicationName,
        time: medicationTime,
        taken: false,
      };
      setMedications([...medications, newMedication]);
    }

    closeModal();
  };

  const removeMedication = (id) => {
    setMedications(medications.filter(med => med.id !== id));
  };

  const toggleTaken = (id) => {
    const updated = medications.map(med =>
      med.id === id ? { ...med, taken: !med.taken } : med
    );
    setMedications(updated);

    const med = medications.find(med => med.id === id);
    if (med && !med.taken) {
      Alert.alert('Remédio tomado', `${med.name} marcado como tomado!`);
    }
  };

  useEffect(() => {
    const interval = setInterval(() => {
      const now = new Date();
      const currentTime = now.getHours().toString().padStart(2, '0') + ':' + now.getMinutes().toString().padStart(2, '0');

      medications.forEach(med => {
        if (med.time === currentTime && !med.taken) {
          Alert.alert('Hora do Remédio', `Hora de tomar: ${med.name}`);
        }
      });
    }, 60000);

    return () => clearInterval(interval);
  }, [medications]);

  return (
    <KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : undefined} style={styles.container}>
      <ScrollView keyboardShouldPersistTaps="handled" contentContainerStyle={styles.scrollContent}>
        <Text style={styles.title}>💊 MédAmigo</Text>

        <TouchableOpacity style={[styles.addButton, { backgroundColor: '#ff4444' }]} onPress={() => navigation.replace('Login')}>
          <Text style={styles.addButtonText}>Sair</Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.addButton} onPress={() => setIsModalVisible(true)}>
          <Text style={styles.addButtonText}>Adicionar Remédio</Text>
        </TouchableOpacity>

        <TouchableOpacity style={[styles.addButton, { backgroundColor: '#555' }]} onPress={() => navigation.navigate('Histórico', { medications })}>
          <Text style={styles.addButtonText}>Ver Histórico</Text>
        </TouchableOpacity>

        <FlatList
          data={medications}
          keyExtractor={item => item.id}
          renderItem={({ item }) => (
            <View style={styles.medItem}>
              <TouchableOpacity style={{ flex: 1 }} onPress={() => openModal(item)}>
                <Text style={{ textDecorationLine: item.taken ? 'line-through' : 'none', color: item.taken ? '#999' : '#333', fontSize: 16 }}>{item.name}</Text>
                <Text style={{ color: '#555' }}>{item.time}</Text>
              </TouchableOpacity>
              <TouchableOpacity onPress={() => toggleTaken(item.id)} style={styles.iconButton}>
                <Image source={{ uri: item.taken ? 'https://cdn-icons-png.flaticon.com/512/1828/1828665.png' : 'https://cdn-icons-png.flaticon.com/512/190/190411.png' }} style={styles.icon} />
              </TouchableOpacity>
              <TouchableOpacity onPress={() => removeMedication(item.id)} style={styles.iconButton}>
                <Image source={{ uri: 'https://cdn-icons-png.flaticon.com/512/1214/1214428.png' }} style={styles.icon} />
              </TouchableOpacity>
            </View>
          )}
          ListEmptyComponent={<Text style={{ textAlign: 'center', marginTop: 20, color: '#777' }}>Nenhum remédio adicionado.</Text>}
        />

        <Modal animationType='slide' transparent={true} visible={isModalVisible} onRequestClose={closeModal}>
          <View style={styles.modalOverlay}>
            <View style={styles.modalContainer}>
              <Text style={styles.modalTitle}>{isEditing ? 'Editar' : 'Novo'} Remédio</Text>
              <TextInput placeholder="Nome do medicamento" value={medicationName} onChangeText={setMedicationName} style={styles.input} />
              <TextInput placeholder="Horário (HH:MM)" value={medicationTime} onChangeText={setMedicationTime} style={styles.input} />
              <TouchableOpacity style={styles.modalButton} onPress={addOrUpdateMedication}>
                <Text style={styles.modalButtonText}>{isEditing ? 'Atualizar' : 'Adicionar'}</Text>
              </TouchableOpacity>
              <TouchableOpacity style={[styles.modalButton, { backgroundColor: '#ccc' }]} onPress={closeModal}>
                <Text style={styles.modalButtonText}>Cancelar</Text>
              </TouchableOpacity>
            </View>
          </View>
        </Modal>
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

// ----------- TELA DE HISTÓRICO -----------
function HistoryScreen({ route }) {
  const { medications } = route.params;

  const getCurrentTime = () => {
    const now = new Date();
    const hours = now.getHours().toString().padStart(2, '0');
    const minutes = now.getMinutes().toString().padStart(2, '0');
    return `${hours}:${minutes}`;
  };

  const currentTime = getCurrentTime();
  const taken = medications.filter(m => m.taken && m.time <= currentTime);
  const pending = medications.filter(m => !m.taken && m.time <= currentTime);

  return (
    <ScrollView contentContainerStyle={styles.container}>
      <Text style={styles.title}>📋 Histórico</Text>

      <Text style={styles.sectionTitle}>✅ Tomados</Text>
      {taken.length === 0 ? <Text style={styles.empty}>Nenhum tomado ainda.</Text> :
        taken.map(m => <Text key={m.id} style={styles.item}>{m.name} às {m.time}</Text>)
      }

      <Text style={styles.sectionTitle}>⏳ Pendentes</Text>
      {pending.length === 0 ? <Text style={styles.empty}>Nenhum pendente.</Text> :
        pending.map(m => <Text key={m.id} style={styles.item}>{m.name} às {m.time}</Text>)
      }
    </ScrollView>
  );
}

// ----------- NAVEGAÇÃO PRINCIPAL -----------
export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Login" screenOptions={{ headerShown: false }}>
        <Stack.Screen name="Login" component={LoginScreen} />
        <Stack.Screen name="Início" component={HomeScreen} />
        <Stack.Screen name="Histórico" component={HistoryScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// ----------- ESTILOS -----------
const styles = StyleSheet.create({
  container: { flexGrow: 1, padding: 20, backgroundColor: '#eaf4f4' },
  scrollContent: { paddingBottom: 100 },
  title: { fontSize: 28, fontWeight: 'bold', textAlign: 'center', marginVertical: 20, color: '#3b5998' },
  addButton: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, marginVertical: 5 },
  addButtonText: { color: '#fff', textAlign: 'center', fontWeight: 'bold' },
  input: { borderColor: '#ccc', borderWidth: 1, borderRadius: 8, padding: 10, marginVertical: 5 },
  medItem: { backgroundColor: '#fff', flexDirection: 'row', alignItems: 'center', padding: 10, marginVertical: 5, borderRadius: 8, elevation: 2 },
  iconButton: { paddingHorizontal: 5 },
  icon: { width: 24, height: 24 },
  modalOverlay: { flex: 1, backgroundColor: 'rgba(0,0,0,0.5)', justifyContent: 'center', alignItems: 'center' },
  modalContainer: { backgroundColor: '#fff', padding: 20, borderRadius: 10, width: '80%' },
  modalTitle: { fontSize: 18, fontWeight: 'bold', marginBottom: 10, textAlign: 'center' },
  modalButton: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, marginTop: 10 },
  modalButtonText: { color: '#fff', textAlign: 'center', fontWeight: 'bold' },
  sectionTitle: { fontSize: 18, fontWeight: 'bold', marginTop: 20, color: '#3b5998' },
  item: { backgroundColor: '#fff', padding: 12, borderRadius: 8, marginTop: 8, elevation: 2 },
  empty: { textAlign: 'center', marginTop: 8, color: '#777' },
});
```

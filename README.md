# Apk-Planetas
import 'package:flutter/material.dart';
import 'views/planet_list_screen.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Gerenciador de Planetas',
      theme: ThemeData(primarySwatch: Colors.deepPurple),
      home: PlanetListScreen(),
    );
  }
}

// models/planet.dart
class Planet {
  final int? id;
  final String nome;
  final double distanciaSol;
  final int tamanho;
  final String? apelido;

  Planet({this.id, required this.nome, required this.distanciaSol, required this.tamanho, this.apelido});

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'nome': nome,
      'distancia_sol': distanciaSol,
      'tamanho': tamanho,
      'apelido': apelido,
    };
  }
}

// services/database_service.dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import '../models/planet.dart';

class DatabaseService {
  static final DatabaseService instance = DatabaseService._init();
  static Database? _database;

  DatabaseService._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('planetas.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(path, version: 1, onCreate: _createDB);
  }

  Future<void> _createDB(Database db, int version) async {
    await db.execute('''
      CREATE TABLE planetas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nome TEXT NOT NULL,
        distancia_sol REAL NOT NULL,
        tamanho INTEGER NOT NULL,
        apelido TEXT
      )
    ''');
  }

  Future<int> addPlanet(Planet planet) async {
    final db = await instance.database;
    return await db.insert('planetas', planet.toMap());
  }

  Future<List<Planet>> getPlanets() async {
    final db = await instance.database;
    final result = await db.query('planetas');
    return result.map((json) => Planet(
      id: json['id'] as int,
      nome: json['nome'] as String,
      distanciaSol: json['distancia_sol'] as double,
      tamanho: json['tamanho'] as int,
      apelido: json['apelido'] as String?,
    )).toList();
  }

  Future<void> deletePlanet(int id) async {
    final db = await instance.database;
    await db.delete('planetas', where: 'id = ?', whereArgs: [id]);
  }
}

// views/planet_list_screen.dart
import 'package:flutter/material.dart';
import '../models/planet.dart';
import '../services/database_service.dart';

class PlanetListScreen extends StatefulWidget {
  @override
  _PlanetListScreenState createState() => _PlanetListScreenState();
}

class _PlanetListScreenState extends State<PlanetListScreen> {
  List<Planet> _planets = [];

  void _loadPlanets() async {
    final planets = await DatabaseService.instance.getPlanets();
    setState(() {
      _planets = planets;
    });
  }

  void _addPlanet() async {
    await DatabaseService.instance.addPlanet(
      Planet(nome: 'Marte', distanciaSol: 1.52, tamanho: 6779, apelido: 'Planeta Vermelho'),
    );
    _loadPlanets();
  }

  void _deletePlanet(int id) async {
    await DatabaseService.instance.deletePlanet(id);
    _loadPlanets();
  }

  @override
  void initState() {
    super.initState();
    _loadPlanets();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Gerenciador de Planetas')),
      body: ListView.builder(
        itemCount: _planets.length,
        itemBuilder: (context, index) {
          final planet = _planets[index];
          return ListTile(
            title: Text(planet.nome),
            subtitle: Text(planet.apelido ?? 'Sem apelido'),
            trailing: IconButton(
              icon: Icon(Icons.delete, color: Colors.red),
              onPressed: () => _deletePlanet(planet.id!),
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: _addPlanet,
      ),
    );
  }
}

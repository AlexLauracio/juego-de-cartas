#include <iostream>
#include <string>
#include <fstream>

using namespace std;

class Configuracion {
private:
    int meta;
    int participantes;
    string palos[4];

public:
    Configuracion() {
        meta = 21;
        participantes = 2;

        palos[0] = "Corazones";
        palos[1] = "Diamantes";
        palos[2] = "Treboles";
        palos[3] = "Espadas";
    }

    bool cargarConfiguracion() {
        ifstream archivo("setup.txt");

        if (!archivo.is_open()) {
            cout << "No se encontro el archivo setup.txt." << endl;
            cout << "Se utilizara la configuracion por defecto." << endl;
            return false;
        }

        string linea;

        while (getline(archivo, linea)) {

            // Quitar salto de linea de Windows (\r) y espacios al final
            while (!linea.empty() &&
                   (linea.back() == '\r' || linea.back() == ' ')) {
                linea.pop_back();
            }

            try {
                if (linea.find("META=") == 0) {
                    meta = stoi(linea.substr(5));
                }
                else if (linea.find("PARTICIPANTES=") == 0) {
                    participantes = stoi(linea.substr(14));
                }
                else if (linea.find("PALO1=") == 0) {
                    palos[0] = linea.substr(6);
                }
                else if (linea.find("PALO2=") == 0) {
                    palos[1] = linea.substr(6);
                }
                else if (linea.find("PALO3=") == 0) {
                    palos[2] = linea.substr(6);
                }
                else if (linea.find("PALO4=") == 0) {
                    palos[3] = linea.substr(6);
                }
            }
            catch (...) {
                cout << "Linea invalida en setup.txt: " << linea << endl;
            }
        }

        archivo.close();

        if (meta < 2) {
            meta = 21;
        }

        if (participantes < 1 || participantes > 10) {
            participantes = 2;
        }

        return true;
    }

    int obtenerMeta() const {
        return meta;
    }

    int obtenerParticipantes() const {
        return participantes;
    }

    string obtenerPalo(int posicion) const {
        if (posicion >= 0 && posicion < 4) {
            return palos[posicion];
        }

        return "Desconocido";
    }

    void mostrarConfiguracion() const {
        cout << "\n=== CONFIGURACION ACTUAL ===" << endl;
        cout << "Puntaje para ganar: " << meta << endl;
        cout << "Participantes: " << participantes << endl;

        cout << "Palos:" << endl;
        cout << "1. " << palos[0] << endl;
        cout << "2. " << palos[1] << endl;
        cout << "3. " << palos[2] << endl;
        cout << "4. " << palos[3] << endl;
    }
};


class GeneradorAleatorio {
private:
    unsigned int semilla;

public:
    GeneradorAleatorio(unsigned int sem = 42) : semilla(sem) {}

    int siguiente(int maximo) {
        semilla = (semilla * 1103515245 + 12345);
        return ((semilla / 65536) % 32768) % maximo;
    }

    void establecerSemilla(unsigned int sem) {
        semilla = sem;
    }

    unsigned int obtenerSemilla() const {
        return semilla;
    }
};


class Carta {
private:
    int valor;
    int palo;

public:
    Carta(int v = 1, int p = 0) {

        if (v >= 1 && v <= 13) {
            valor = v;
        }
        else {
            valor = 1;
        }

        if (p >= 0 && p < 4) {
            palo = p;
        }
        else {
            palo = 0;
        }
    }

    int obtenerValor() const {
        return valor;
    }

    int obtenerPalo() const {
        return palo;
    }

    string obtenerNombrePalo(const Configuracion& configuracion) const {
        return configuracion.obtenerPalo(palo);
    }

    string obtenerDescripcion(const Configuracion& configuracion) const {

        string vStr;

        if (valor == 1) {
            vStr = "As";
        }
        else if (valor == 11) {
            vStr = "J";
        }
        else if (valor == 12) {
            vStr = "Q";
        }
        else if (valor == 13) {
            vStr = "K";
        }
        else {
            vStr = to_string(valor);
        }

        return vStr + " de " + obtenerNombrePalo(configuracion);
    }
};


class Mazo {
private:
    Carta cartas[52];
    int indice;
    GeneradorAleatorio generador;

public:
    Mazo() : indice(0) {

        for (int i = 0; i < 52; i++) {
            cartas[i] = Carta((i % 13) + 1, i / 13);
        }

        barajar();
    }

    void barajar() {

        generador.establecerSemilla(
            generador.obtenerSemilla() + 13
        );

        for (int i = 0; i < 52; i++) {

            int j = generador.siguiente(52);

            Carta temp = cartas[i];
            cartas[i] = cartas[j];
            cartas[j] = temp;
        }

        indice = 0;
    }

    Carta repartir() {

        if (indice < 52) {
            return cartas[indice++];
        }

        return Carta(1, 0);
    }

    void mostrarMazo(const Configuracion& configuracion) const {

        cout << "\n=== ESTADO DEL MAZO ===" << endl;
        cout << "Cartas restantes: " << (52 - indice) << endl;

        for (int i = indice; i < 52; i++) {
            cout << "[" << i << "] "
                 << cartas[i].obtenerDescripcion(configuracion)
                 << endl;
        }
    }

    int obtenerIndice() const {
        return indice;
    }

    void establecerIndice(int idx) {
        indice = idx;
    }

    void establecerSemilla(unsigned int sem) {
        generador.establecerSemilla(sem);
    }

    unsigned int obtenerSemilla() const {
        return generador.obtenerSemilla();
    }
};


class Mano {
private:
    int suma;

public:
    Mano() : suma(0) {}

    void agregarCarta(const Carta& carta) {
        suma += carta.obtenerValor();
    }

    int obtenerSuma() const {
        return suma;
    }

    void establecerSuma(int s) {
        suma = s;
    }

    void reiniciar() {
        suma = 0;
    }
};


class Participante {
protected:
    Mano mano;

public:
    virtual void recibirCarta(const Carta& carta) {
        mano.agregarCarta(carta);
    }

    int obtenerTotalPuntos() const {
        return mano.obtenerSuma();
    }

    void establecerTotalPuntos(int suma) {
        mano.establecerSuma(suma);
    }

    void reiniciarMano() {
        mano.reiniciar();
    }

    virtual ~Participante() {}
};


class Jugador : public Participante {
private:
    string nombre;

public:
    Jugador(string n = "Jugador") {
        nombre = n;
    }

    string obtenerNombre() const {
        return nombre;
    }

    void establecerNombre(string n) {
        nombre = n;
    }
};


class GestorPartida {
private:
    Mazo mazo;
    Configuracion configuracion;

    Jugador jugadores[10];

    int cantidadParticipantes;

public:

    GestorPartida() {

        configuracion.cargarConfiguracion();

        cantidadParticipantes =
            configuracion.obtenerParticipantes();

        for (int i = 0; i < cantidadParticipantes; i++) {

            jugadores[i].establecerNombre(
                "Jugador " + to_string(i + 1)
            );
        }
    }

    Mazo& obtenerMazo() {
        return mazo;
    }

    Configuracion& obtenerConfiguracion() {
        return configuracion;
    }

    void iniciar(bool esContinuacion) {

        if (!esContinuacion) {

            configuracion.cargarConfiguracion();

            cantidadParticipantes =
                configuracion.obtenerParticipantes();

            for (int i = 0; i < cantidadParticipantes; i++) {

                jugadores[i].reiniciarMano();

                jugadores[i].establecerNombre(
                    "Jugador " + to_string(i + 1)
                );
            }

            mazo.barajar();

            cout << "\n=== NUEVA PARTIDA ===" << endl;

            configuracion.mostrarConfiguracion();
        }
        else {

            cout << "\n=== REANUDANDO PARTIDA ===" << endl;
        }


        for (int i = 0; i < cantidadParticipantes; i++) {

            int opcionJuego = 1;

            cout << "\n================================" << endl;
            cout << "        TURNO DE "
                 << jugadores[i].obtenerNombre()
                 << endl;
            cout << "================================" << endl;

            while (
                opcionJuego == 1 &&
                jugadores[i].obtenerTotalPuntos()
                    <= configuracion.obtenerMeta()
            ) {

                cout << "\nTu suma actual: "
                     << jugadores[i].obtenerTotalPuntos()
                     << endl;

                cout << "Meta para ganar: "
                     << configuracion.obtenerMeta()
                     << endl;

                cout << "¿Deseas pedir carta? "
                     << "(1: Si / 0: Plantarse / 2: Guardar y Salir): ";

                cin >> opcionJuego;


                if (opcionJuego == 2) {

                    guardarEnArchivo();

                    return;
                }


                if (opcionJuego == 1) {

                    Carta cartaJugador = mazo.repartir();

                    jugadores[i].recibirCarta(cartaJugador);

                    cout << "Tu carta: "
                         << cartaJugador.obtenerDescripcion(configuracion)
                         << " | Tu suma total: "
                         << jugadores[i].obtenerTotalPuntos()
                         << endl;


                    if (
                        jugadores[i].obtenerTotalPuntos()
                        > configuracion.obtenerMeta()
                    ) {

                        cout << "Te pasaste de la meta." << endl;
                        break;
                    }
                }
            }

            if (
                jugadores[i].obtenerTotalPuntos()
                <= configuracion.obtenerMeta()
            ) {

                cout << "\n"
                     << jugadores[i].obtenerNombre()
                     << " termino con "
                     << jugadores[i].obtenerTotalPuntos()
                     << " puntos."
                     << endl;
            }
        }

        evaluarResultados();
    }


    void guardarEnArchivo() {

        ofstream archivo("partida_juego21.txt");

        if (archivo.is_open()) {

            archivo << configuracion.obtenerMeta()
                    << endl;

            archivo << cantidadParticipantes
                    << endl;

            for (int i = 0; i < cantidadParticipantes; i++) {

                archivo << jugadores[i].obtenerTotalPuntos()
                        << endl;
            }

            archivo << mazo.obtenerIndice()
                    << endl;

            archivo << mazo.obtenerSemilla()
                    << endl;

            archivo.close();

            cout << "¡Partida guardada exitosamente!" << endl;
        }
        else {

            cout << "Error al abrir el archivo para guardar."
                 << endl;
        }
    }


    bool cargarDesdeArchivo() {

        ifstream archivo("partida_juego21.txt");

        if (archivo.is_open()) {

            int metaGuardada;
            int participantesGuardados;
            int indice;

            unsigned int semilla;

            archivo >> metaGuardada;
            archivo >> participantesGuardados;

            cantidadParticipantes =
                participantesGuardados;

            configuracion.cargarConfiguracion();

            configuracion.mostrarConfiguracion();

            for (int i = 0;
                 i < cantidadParticipantes;
                 i++) {

                int suma;

                archivo >> suma;

                jugadores[i].establecerNombre(
                    "Jugador " + to_string(i + 1)
                );

                jugadores[i].establecerTotalPuntos(suma);
            }

            archivo >> indice;
            archivo >> semilla;

            archivo.close();

            mazo.establecerIndice(indice);
            mazo.establecerSemilla(semilla);

            cout << "\n¡Partida recuperada exitosamente!"
                 << endl;

            iniciar(true);

            return true;
        }

        cout << "No se encontro ningun archivo de partida guardada."
             << endl;

        return false;
    }


    void evaluarResultados() {

        int mejorPuntaje = -1;
        int ganador = -1;

        cout << "\n================================" << endl;
        cout << "          RESULTADOS" << endl;
        cout << "================================" << endl;


        for (int i = 0; i < cantidadParticipantes; i++) {

            int puntaje =
                jugadores[i].obtenerTotalPuntos();

            cout << jugadores[i].obtenerNombre()
                 << ": "
                 << puntaje
                 << " puntos"
                 << endl;


            if (
                puntaje <= configuracion.obtenerMeta() &&
                puntaje > mejorPuntaje
            ) {

                mejorPuntaje = puntaje;
                ganador = i;
            }
        }


        cout << "\nMeta: "
             << configuracion.obtenerMeta()
             << endl;


        if (ganador == -1) {

            cout << "NINGUN PARTICIPANTE GANO."
                 << endl;
        }
        else {

            cout << "GANADOR: "
                 << jugadores[ganador].obtenerNombre()
                 << endl;

            cout << "Puntaje: "
                 << mejorPuntaje
                 << endl;
        }
    }
};


class MenuJuego {
private:
    GestorPartida gestor;

public:

    void desplegar() {

        int opcionMenu = 0;

        do {

            cout << "\n======================="
                 << endl;

            cout << "   JUEGO CONFIGURABLE"
                 << endl;

            cout << "======================="
                 << endl;

            cout << "1. Nueva Partida" << endl;
            cout << "2. Cargar Partida Guardada" << endl;
            cout << "3. Ver y Barajar el Mazo" << endl;
            cout << "4. Ver Configuracion" << endl;
            cout << "5. Salir" << endl;

            cout << "Seleccione una opcion: ";

            cin >> opcionMenu;


            if (opcionMenu == 1) {

                gestor.iniciar(false);
            }

            else if (opcionMenu == 2) {

                gestor.cargarDesdeArchivo();
            }

            else if (opcionMenu == 3) {

                int subOpcion;

                do {

                    gestor.obtenerMazo()
                        .mostrarMazo(
                            gestor.obtenerConfiguracion()
                        );

                    cout << "\n--- OPCIONES DE MAZO ---"
                         << endl;

                    cout << "1. Volver a barajar"
                         << endl;

                    cout << "2. Regresar al menu principal"
                         << endl;

                    cout << "Seleccione una opcion: ";

                    cin >> subOpcion;


                    if (subOpcion == 1) {

                        gestor.obtenerMazo().barajar();

                        cout << "¡Mazo barajado exitosamente!"
                             << endl;
                    }

                } while (subOpcion != 2);
            }

            else if (opcionMenu == 4) {

                gestor.obtenerConfiguracion()
                    .cargarConfiguracion();

                gestor.obtenerConfiguracion()
                    .mostrarConfiguracion();
            }

            else if (opcionMenu == 5) {

                cout << "Saliendo del juego..."
                     << endl;
            }

            else {

                cout << "Opcion invalida."
                     << endl;
            }

        } while (opcionMenu != 5);
    }
};


int main() {

    MenuJuego menu;

    menu.desplegar();

    return 0;
}

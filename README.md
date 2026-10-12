
import csv
from pathlib import Path
from datetime import date

CARPETA = Path("datos_academicos")
CARPETA.mkdir(exist_ok=True)

ARCHIVOS = {
    "docentes": ["id", "nombres", "apellidos", "correo"],
    "cursos": ["id", "curso", "paralelo", "periodo"],
    "materias": ["id", "materia", "id_curso", "id_docente"],
    "estudiantes": ["id", "nombres", "apellidos", "id_curso"],
    "asistencias": [
        "fecha", "id_estudiante", "id_materia", "estado", "observacion"
    ],
    "notas": [
        "fecha", "id_estudiante", "id_materia",
        "actividad", "nota", "sobre"
    ],
    "recuperaciones": [
        "fecha", "id_estudiante", "id_materia",
        "nota_original", "nota_recuperacion", "nota_final"
    ],
}


def leer(tabla):
    archivo = CARPETA / f"{tabla}.csv"
    if not archivo.exists():
        with open(archivo, "w", newline="", encoding="utf-8-sig") as f:
            csv.DictWriter(f, fieldnames=ARCHIVOS[tabla]).writeheader()

    with open(archivo, newline="", encoding="utf-8-sig") as f:
        return list(csv.DictReader(f))


def guardar(tabla, filas):
    with open(CARPETA / f"{tabla}.csv", "w",
              newline="", encoding="utf-8-sig") as f:
        escritor = csv.DictWriter(f, fieldnames=ARCHIVOS[tabla])
        escritor.writeheader()
        escritor.writerows(filas)


def agregar(tabla, registro):
    filas = leer(tabla)
    filas.append(registro)
    guardar(tabla, filas)


def siguiente_id(tabla):
    filas = leer(tabla)
    numeros = []
    for fila in filas:
        try:
            numeros.append(int(fila["id"]))
        except (ValueError, KeyError):
            pass
    return str(max(numeros, default=0) + 1)


def pedir(mensaje):
    while True:
        valor = input(mensaje).strip()
        if valor:
            return valor
        print("Este campo es obligatorio.")


def elegir(tabla, titulo):
    filas = leer(tabla)
    if not filas:
        print(f"No hay registros en {tabla}.")
        return None

    print(f"\n--- {titulo} ---")
    for fila in filas:
        print(" | ".join(f"{k}: {v}" for k, v in fila.items()))

    identificador = pedir("Escribe el ID: ")
    for fila in filas:
        if fila.get("id") == identificador:
            return fila

    print("No se encontró ese ID.")
    return None


def registrar_docente():
    print("\nREGISTRAR DOCENTE")
    agregar("docentes", {
        "id": siguiente_id("docentes"),
        "nombres": pedir("Nombres: "),
        "apellidos": pedir("Apellidos: "),
        "correo": input("Correo (opcional): ").strip()
    })
    print("Docente registrado.")


def registrar_curso():
    print("\nREGISTRAR CURSO")
    agregar("cursos", {
        "id": siguiente_id("cursos"),
        "curso": pedir("Nombre del curso: "),
        "paralelo": pedir("Paralelo: "),
        "periodo": pedir("Periodo lectivo: ")
    })
    print("Curso registrado.")


def registrar_materia():
    print("\nREGISTRAR MATERIA")
    curso = elegir("cursos", "CURSOS")
    if not curso:
        return

    docente = elegir("docentes", "DOCENTES")
    if not docente:
        return

    agregar("materias", {
        "id": siguiente_id("materias"),
        "materia": pedir("Nombre de la materia: "),
        "id_curso": curso["id"],
        "id_docente": docente["id"]
    })
    print("Materia registrada.")


def registrar_estudiante():
    print("\nREGISTRAR ESTUDIANTE")
    curso = elegir("cursos", "CURSOS")
    if not curso:
        return

    identificador = pedir("Código único del estudiante: ")
    if any(e["id"] == identificador for e in leer("estudiantes")):
        print("Ese código ya está registrado.")
        return

    agregar("estudiantes", {
        "id": identificador,
        "nombres": pedir("Nombres: "),
        "apellidos": pedir("Apellidos: "),
        "id_curso": curso["id"]
    })
    print("Estudiante registrado.")


def elegir_materia():
    filas = leer("materias")
    if not filas:
        print("Primero registra una materia.")
        return None

    for m in filas:
        print(
            f'ID {m["id"]}: {m["materia"]} '
            f'(curso ID {m["id_curso"]}, '
            f'docente ID {m["id_docente"]})'
        )

    identificador = pedir("ID de la materia: ")
    for materia in filas:
        if materia["id"] == identificador:
            return materia

    print("Materia no encontrada.")
    return None


def estudiantes_del_curso(id_curso):
    return [
        e for e in leer("estudiantes")
        if e["id_curso"] == id_curso
    ]


def elegir_estudiante(id_curso):
    estudiantes = estudiantes_del_curso(id_curso)
    if not estudiantes:
        print("No hay estudiantes en ese curso.")
        return None

    for e in estudiantes:
        print(f'{e["id"]}: {e["apellidos"]}, {e["nombres"]}')

    identificador = pedir("Código del estudiante: ")
    for e in estudiantes:
        if e["id"] == identificador:
            return e

    print("Estudiante no encontrado en este curso.")
    return None


def registrar_asistencia():
    print("\nREGISTRO DE ASISTENCIA")
    materia = elegir_materia()
    if not materia:
        return

    estudiantes = estudiantes_del_curso(materia["id_curso"])
    if not estudiantes:
        print("Este curso no tiene estudiantes registrados.")
        return

    fecha = input(
        f"Fecha AAAA-MM-DD (Enter = {date.today()}): "
    ).strip() or str(date.today())

    for e in estudiantes:
        print(f'\n{e["id"]} - {e["apellidos"]}, {e["nombres"]}')
        print("P = Presente | F = Falta | A = Atraso | U = Fuga")

        while True:
            estado = input("Estado (Enter = P): ").strip().upper() or "P"
            if estado in ("P", "F", "A", "U"):
                break
            print("Opción incorrecta.")

        observacion = input("Observación (opcional): ").strip()
        filas = leer("asistencias")

        existente = next((
            r for r in filas
            if r["fecha"] == fecha
            and r["id_estudiante"] == e["id"]
            and r["id_materia"] == materia["id"]
        ), None)

        if existente:
            existente["estado"] = estado
            existente["observacion"] = observacion
            guardar("asistencias", filas)
        else:
            agregar("asistencias", {
                "fecha": fecha,
                "id_estudiante": e["id"],
                "id_materia": materia["id"],
                "estado": estado,
                "observacion": observacion
            })

    print("Asistencia guardada correctamente.")


def pedir_nota(mensaje, maximo=None):
    while True:
        try:
            valor = float(pedir(mensaje).replace(",", "."))
            if valor < 0 or (maximo is not None and valor > maximo):
                print("La nota está fuera del rango permitido.")
                continue
            return valor
        except ValueError:
            print("Escribe un número válido.")


def registrar_nota():
    print("\nREGISTRAR NOTA")
    materia = elegir_materia()
    if not materia:
        return

    estudiante = elegir_estudiante(materia["id_curso"])
    if not estudiante:
        return

    actividad = pedir("Actividad (tarea, prueba, examen): ")
    sobre = pedir_nota("Nota máxima (ej. 10): ")
    if sobre <= 0:
        print("La nota máxima debe ser mayor que cero.")
        return

    nota = pedir_nota("Nota obtenida: ", sobre)
    agregar("notas", {
        "fecha": str(date.today()),
        "id_estudiante": estudiante["id"],
        "id_materia": materia["id"],
        "actividad": actividad,
        "nota": str(nota),
        "sobre": str(sobre)
    })
    print("Nota registrada.")


def registrar_recuperacion():
    print("\nREGISTRAR RECUPERACIÓN")
    materia = elegir_materia()
    if not materia:
        return

    estudiante = elegir_estudiante(materia["id_curso"])
    if not estudiante:
        return

    sobre = pedir_nota("Nota máxima (ej. 10): ")
    if sobre <= 0:
        print("La nota máxima debe ser mayor que cero.")
        return

    original = pedir_nota("Nota original: ", sobre)
    recuperacion = pedir_nota("Nota de recuperación: ", sobre)

    # Regla predeterminada: conservar la nota más alta.
    final = max(original, recuperacion)

    agregar("recuperaciones", {
        "fecha": str(date.today()),
        "id_estudiante": estudiante["id"],
        "id_materia": materia["id"],
        "nota_original": str(original),
        "nota_recuperacion": str(recuperacion),
        "nota_final": str(final)
    })
    print(f"Recuperación registrada. Nota final: {final}/{sobre}")


def reportes():
    while True:
        print("\n--- REPORTES ---")
        print("1. Docentes")
        print("2. Cursos")
        print("3. Materias")
        print("4. Estudiantes")
        print("5. Asistencia, faltas y fugas")
        print("6. Notas")
        print("7. Recuperaciones")
        print("0. Volver")

        opcion = input("Opción: ").strip()
        tablas = {
            "1": "docentes",
            "2": "cursos",
            "3": "materias",
            "4": "estudiantes",
            "6": "notas",
            "7": "recuperaciones"
        }

        if opcion in tablas:
            filas = leer(tablas[opcion])
            if filas:
                for fila in filas:
                    print(" | ".join(f"{k}: {v}" for k, v in fila.items()))
            else:
                print("No hay registros.")

        elif opcion == "5":
            materia = elegir_materia()
            if not materia:
                continue

            registros = [
                a for a in leer("asistencias")
                if a["id_materia"] == materia["id"]
            ]

            for e in estudiantes_del_curso(materia["id_curso"]):
                propios = [
                    a for a in registros
                    if a["id_estudiante"] == e["id"]
                ]
                conteo = {
                    estado: sum(1 for a in propios if a["estado"] == estado)
                    for estado in ("P", "F", "A", "U")
                }
                total = len(propios)
                porcentaje = (
                    (conteo["P"] + conteo["A"]) / total * 100
                    if total else 0
                )
                print(
                    f'{e["id"]} - {e["apellidos"]}, {e["nombres"]} | '
                    f'Presentes: {conteo["P"]} | Faltas: {conteo["F"]} | '
                    f'Atrasos: {conteo["A"]} | Fugas: {conteo["U"]} | '
                    f'Asistencia: {porcentaje:.1f}%'
                )

        elif opcion == "0":
            break
        else:
            print("Opción incorrecta.")


def menu():
    # Crear todos los CSV al iniciar el programa.
    for tabla in ARCHIVOS:
        leer(tabla)

    while True:
        print("\n" + "=" * 45)
        print("     SISTEMA DE REGISTRO ACADÉMICO")
        print("=" * 45)
        print("1. Registrar docente")
        print("2. Registrar curso")
        print("3. Registrar materia")
        print("4. Registrar estudiante")
        print("5. Registrar asistencia / falta / fuga")
        print("6. Registrar nota")
        print("7. Registrar recuperación")
        print("8. Consultar reportes")
        print("0. Salir")

        opcion = input("Selecciona una opción: ").strip()

        opciones = {
            "1": registrar_docente,
            "2": registrar_curso,
            "3": registrar_materia,
            "4": registrar_estudiante,
            "5": registrar_asistencia,
            "6": registrar_nota,
            "7": registrar_recuperacion,
            "8": reportes
        }

        if opcion == "0":
            print("Datos guardados. Hasta pronto.")
            break
        elif opcion in opciones:
            opciones[opcion]()
        else:
            print("Opción incorrecta.")


if __name__ == "__main__":
    try:
        menu()
    except KeyboardInterrupt:
        print("\nPrograma detenido. Los datos guardados se conservan.")

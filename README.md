import speech_recognition as sr
import pyttsx3
import threading
import flet as ft
import numpy as np
import matplotlib
import math
matplotlib.use('Agg')  # Configurar matplotlib para usar el backend 'Agg'
# para generar gráficos en formato de imagen sin necesidad de una ventana de visualización.
import matplotlib.pyplot as plt
import base64
import io

# Inicializando variables de reconocimiento de voz y síntesis de voz
r = sr.Recognizer()
engine = pyttsx3.init()

# Bloqueo para manejar el motor de voz en hilos
lock = threading.Lock()

# Función para reiniciar el motor de voz
def restart_engine():
    global engine
    with lock:
        engine.stop()  # Detiene cualquier proceso en curso
        del engine
        engine = pyttsx3.init()

# Función para hablar el texto con pyttsx3
def speak(text):
    try:
        with lock:
            restart_engine()  # Reiniciar el motor antes de cada uso
            print(f"Pronunciando: {text}")  # Depuración
            engine.say(text)
            engine.runAndWait()
    except Exception as e:
        print(f"Error al hablar: {e}")  # Manejar errores
        restart_engine()  # Reiniciar el motor si ocurre un error

# Función para convertir un número a formato hablado (manejo de números grandes y pequeños)
def convert_number_to_speech(num):
    try:
        num = float(num)
        if abs(num) >= 1e6:
            return f"{num:,.0f}"
        elif abs(num) < 1:
            return "{:.10f}".format(num).rstrip('0').rstrip('.')
        else:
            return str(num)
    except ValueError:
        return str(num)

# Función para reconocer y calcular el resultado
class MicrofonoTab(ft.Container):
    def __init__(self):
        super().__init__()
        self.text_display = ft.Text(
            value="Pulsa en el icono de micrófono para\n iniciar y di la operación matemática\n que quieres resolver",
            size=20,
            font_family='Georgia',
            text_align=ft.TextAlign.CENTER
        )
        self.microfono_button = ft.IconButton(
            icon=ft.icons.MIC,
            icon_size=40,
            on_click=self.on_microfono_click,
            style=ft.ButtonStyle(
                color=ft.colors.PURPLE,
            )
        )
        self.content = ft.Container(
            content=ft.Column(
                controls=[
                    ft.Row(
                        controls=[
                            self.text_display,
                        ],
                        alignment=ft.MainAxisAlignment.CENTER  # Alineación centrada en la fila
                    ),
                    self.microfono_button
                ],
                alignment=ft.MainAxisAlignment.CENTER,
                horizontal_alignment=ft.CrossAxisAlignment.CENTER
            ),
            alignment=ft.alignment.center,
            height=400,
            width=300,
        )
        self.listening = False

    def on_microfono_click(self, e):
        if not self.listening:
            self.text_display.value = "ESCUCHANDO..."
            self.text_display.update()
            # Iniciar un nuevo hilo para el reconocimiento y cálculo
            threading.Thread(target=self.recognize_and_calculate).start()
            self.listening = True
            self.microfono_button.icon = ft.icons.MIC
        else:
            self.text_display.value = "Pulsa en el icono de micrófono para\n iniciar y di la operación matemática\n que quieres resolver"
            self.text_display.update()
            self.microfono_button.icon = ft.icons.MIC
            self.listening = False

    def recognize_and_calculate(self):
        try:
            with sr.Microphone() as source:
                print("Escuchando...")  # Depuración
                audio = r.listen(source)
                equation = r.recognize_google(audio, language='es-ES').lower()
                equation = equation.replace("á", "a").replace("é", "e").replace("í", "i").replace("ó", "o").replace("ú", "u")
                equation = equation.replace("*", "x").replace("dividido por", "/").replace("entre", "/")
                print(f"Operación reconocida: {equation}")  # Depuración

                # Asegúrate de que sm.getResult está correctamente definido para evaluar la operación
                res = sm.getResult(equation)
                result_text = f"Operación reconocida: {equation}\nResultado: {res}"
                self.text_display.value = result_text
                self.text_display.update()

                speak_result = convert_number_to_speech(res)
                speak(f" {speak_result}")

        except sr.UnknownValueError:
            self.text_display.value = "No se pudo entender el audio."
            self.text_display.update()
        except sr.RequestError as e:
            self.text_display.value = f"Error al solicitar resultados: {e}"
            self.text_display.update()
        except Exception as e:
            print(f"Error en recognize_and_calculate: {e}")  # Depuración
        finally:
            self.text_display.value = "Clica en el icono de micrófono para\n iniciar y di la operación matemática\n que quieres resolver"
            self.text_display.update()
            self.listening = False
            self.microfono_button.icon = ft.icons.MIC_EXTERNAL_ON_OUTLINED
            self.microfono_button.update()

class GraficosTab(ft.Container):
    def __init__(self):
        super().__init__()

        # Funciones para generar gráficos
        def plot_line(a, b):
            x = np.linspace(-10, 10, 400)
            y = a * x + b

            fig, ax = plt.subplots(figsize=(5, 3))  # Ajustar el tamaño del gráfico
            ax.plot(x, y, label=f'y = {a}x + {b}')
            ax.set_xlabel('x')
            ax.set_ylabel('y')
            ax.set_title('Gráfico de la Ecuación Lineal')
            ax.legend()

            # Guardar el gráfico en un buffer de bytes que es  es una región de memoria
            # que almacena datos en forma de bytes
            buf = io.BytesIO()
            plt.savefig(buf, format='png', bbox_inches='tight')  # Ajustar el ajuste del gráfico
            buf.seek(0)
            plt.close(fig)  # Cerrar la figura para liberar recursos
            return buf.getvalue()

        def plot_quadratic(a, b, c):
            x = np.linspace(-10, 10, 400)
            y = a * x ** 2 + b * x + c

            fig, ax = plt.subplots(figsize=(5, 3))  # Ajustar el tamaño del gráfico
            ax.plot(x, y, label=f'y = {a}x² + {b}x + {c}')
            ax.set_xlabel('x')
            ax.set_ylabel('y')
            ax.set_title('Gráfico de la Ecuación Cuadrática')
            ax.legend()

            # Guardar el gráfico en un buffer de bytes
            buf = io.BytesIO()
            plt.savefig(buf, format='png', bbox_inches='tight')  # Ajustar el ajuste del gráfico
            buf.seek(0)
            plt.close(fig)  # Cerrar la figura para liberar recursos
            return buf.getvalue()

        # Inputs y vistas para la ecuación lineal
        a_input_linear = ft.TextField(label="Coeficiente a", width=200, keyboard_type=ft.KeyboardType.NUMBER)
        b_input_linear = ft.TextField(label="Coeficiente b", width=200, keyboard_type=ft.KeyboardType.NUMBER)
        image_linear = ft.Image(width=500, height=300)  # Ajustar el tamaño del componente de imagen

        def on_plot_linear_click(e):
            try:
                a = float(a_input_linear.value)
                b = float(b_input_linear.value)
                img_data = plot_line(a, b)
                image_linear.src_base64 = base64.b64encode(img_data).decode()
                show_only_graph(image_linear, is_quadratic=False)
            except ValueError:
                image_linear.src_base64 = None  # Si se produce un error, no se asigna la imagen
            self.update()

        plot_button_linear = ft.ElevatedButton(text="Generar Gráfico Lineal", on_click=on_plot_linear_click)

        linear_view = ft.Column(
            controls=[
                a_input_linear,
                b_input_linear,
                plot_button_linear
            ],
            visible=True,
            alignment=ft.MainAxisAlignment.CENTER  # Centrar los componentes verticalmente
        )

        # Inputs y vistas para la ecuación cuadrática
        a_input_quad = ft.TextField(label="Coeficiente a", width=200, keyboard_type=ft.KeyboardType.NUMBER)
        b_input_quad = ft.TextField(label="Coeficiente b", width=200, keyboard_type=ft.KeyboardType.NUMBER)
        c_input_quad = ft.TextField(label="Coeficiente c", width=200, keyboard_type=ft.KeyboardType.NUMBER)
        image_quad = ft.Image(width=500, height=300)  # Ajustar el tamaño del componente de imagen

        def on_plot_quad_click(e):
            try:
                a = float(a_input_quad.value)
                b = float(b_input_quad.value)
                c = float(c_input_quad.value)
                img_data = plot_quadratic(a, b, c)
                image_quad.src_base64 = base64.b64encode(img_data).decode()
                show_only_graph(image_quad, is_quadratic=True)
            except ValueError:
                image_quad.src_base64 = None  # Si hay error, no se asigna la imagen
            self.update()

        plot_button_quad = ft.ElevatedButton(text="Generar Gráfico Cuadrático", on_click=on_plot_quad_click)

        quadratic_view = ft.Column(
            controls=[
                a_input_quad,
                b_input_quad,
                c_input_quad,
                plot_button_quad
            ],
            visible=False,
            alignment=ft.MainAxisAlignment.CENTER
        )

        # Función para mostrar solo el gráfico y la cruz negra
        def show_only_graph(image, is_quadratic):
            # Ocultar todos los controles de la vista principal
            content.visible = False
            # Mostrar solo el gráfico y la cruz para retroceder
            graph_view.controls = [
                ft.Container(
                    content=image,
                    alignment=ft.alignment.center,
                    expand=True
                ),
                ft.IconButton(icon=ft.icons.CLOSE, icon_color="black", on_click=show_main_view, alignment=ft.alignment.top_right)
            ]
            graph_view.visible = True
            self.update()

        # Función para volver a la vista principal
        def show_main_view(e):
            graph_view.visible = False
            content.visible = True
            self.update()

        # Botones para seleccionar tipo de gráfico
        def show_linear_view(e):
            linear_view.visible = True
            quadratic_view.visible = False
            self.update()

        def show_quadratic_view(e):
            linear_view.visible = False
            quadratic_view.visible = True
            self.update()

        linear_button = ft.ElevatedButton(text="Ecuación Lineal", on_click=show_linear_view)
        quadratic_button = ft.ElevatedButton(text="Ecuación Cuadrática", on_click=show_quadratic_view)

        # Vista principal
        content = ft.Column(
            controls=[
                ft.Text(value="Escoge el tipo de ecuación:", size=24, weight=ft.FontWeight.BOLD, font_family="Georgia"),
                ft.Container(height=20),  # Espacio entre el texto y los botones
                ft.Row(
                    controls=[
                        linear_button,
                        quadratic_button
                    ],
                    alignment=ft.MainAxisAlignment.CENTER
                ),
                linear_view,
                quadratic_view
            ],
            alignment=ft.MainAxisAlignment.CENTER
        )

        # Vista solo para mostrar el gráfico con la cruz
        graph_view = ft.Stack(
            controls=[],
            visible=False,  # Esta vista es inicialmente invisible
            expand=True
        )

        # Contenedor principal
        self.content = ft.Stack(
            controls=[content, graph_view],  # Ambos stacks, solo uno será visible a la vez
            expand=True
        )

# Clase para los botones de la calculadora
class CalcButton(ft.ElevatedButton):
    def __init__(self, name, text, button_clicked, expand=3):
        # Parámetro de name: para identificar nombre del botón
        # Parámetro de text: texto que muestra el botón
        # Parámetro de button_clicked: función que se ejecuta cuando el botón es clicado
        # Parámetro de expand: factor de expansión del botón
        super().__init__()
        self.text = text
        self.expand = expand
        self.on_click = button_clicked
        self.data = text
        self.name = name

    def display_name(self):
        return f"XApp name: {self.name}"

# Clase para los botones numéricos de la calculadora
class DigitButton(CalcButton):
    def __init__(self, text, button_clicked, expand=3):
        super().__init__(name="DigitButton", text=text, button_clicked=button_clicked, expand=expand)
        self.bgcolor = ft.colors.WHITE24   # color de fondo del botón
        self.color = ft.colors.WHITE       #  color del texto del botón

# Clase para los botones de acción (operaciones) de la calculadora
class ActionButton(CalcButton):
    def __init__(self, text, button_clicked):
        super().__init__(name="ActionButton", text=text, button_clicked=button_clicked)
        self.bgcolor = ft.colors.PURPLE
        self.color = ft.colors.WHITE

# Clase para los botones de acción adicionales de la calculadora (como paréntesis)
class ExtraActionButton(CalcButton):
    def __init__(self, text, button_clicked):
        super().__init__(name="ExtraActionButton", text=text, button_clicked=button_clicked)
        self.bgcolor = ft.colors.PURPLE
        self.color = ft.colors.WHITE
        self.margin = ft.margin.only(bottom= 9)

class CalculatorApp(ft.Container):
    def __init__(self):
        super().__init__()
        self.reset()

        self.result = ft.Text(value="0", color=ft.colors.WHITE, size=30)
        self.width = 700
        self.bgcolor = ft.colors.PURPLE_100
        self.border_radius = ft.border_radius.all(40)
        self.padding = 30  # espacio interno que se agrega alrededor del contenido dentro de un contenedor
        self.margin = ft.margin.only(top=13, bottom=13)  # Añadir margen superior e inferior
        self.content = ft.Column(
            controls=[
                ft.Row(controls=[self.result], alignment=ft.MainAxisAlignment.END),
                ft.Row(
                    controls=[
                        ExtraActionButton(text="(", button_clicked=self.button_clicked),
                        ExtraActionButton(text=")", button_clicked=self.button_clicked),
                        ExtraActionButton(text="[", button_clicked=self.button_clicked),
                        ExtraActionButton(text="]", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
                ft.Row(
                    controls=[
                        ExtraActionButton(text="AC", button_clicked=self.button_clicked),
                        ExtraActionButton(text="+/-", button_clicked=self.button_clicked),
                        ExtraActionButton(text="%", button_clicked=self.button_clicked),
                        ActionButton(text="÷", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
                ft.Row(
                    controls=[
                        DigitButton(text="7", button_clicked=self.button_clicked),
                        DigitButton(text="8", button_clicked=self.button_clicked),
                        DigitButton(text="9", button_clicked=self.button_clicked),
                        ActionButton(text="×", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
                ft.Row(
                    controls=[
                        DigitButton(text="4", button_clicked=self.button_clicked),
                        DigitButton(text="5", button_clicked=self.button_clicked),
                        DigitButton(text="6", button_clicked=self.button_clicked),
                        ActionButton(text="-", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
                ft.Row(
                    controls=[
                        DigitButton(text="1", button_clicked=self.button_clicked),
                        DigitButton(text="2", button_clicked=self.button_clicked),
                        DigitButton(text="3", button_clicked=self.button_clicked),
                        ActionButton(text="+", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
                ft.Row(
                    controls=[
                        DigitButton(text="0", expand=6, button_clicked=self.button_clicked),
                        DigitButton(text=".", button_clicked=self.button_clicked),
                        ActionButton(text="=", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
                ft.Row(
                    controls=[
                        ExtraActionButton(text="√", button_clicked=self.button_clicked),
                        ExtraActionButton(text="x²", button_clicked=self.button_clicked),
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                ),
            ],
            spacing=10,
        )

    def button_clicked(self, e):
        data = e.control.data
        print(f"Button clicked with data = {data}")

        # Procesar los diferentes tipos de botones
        if data == "AC":
            self.result.value = "0"   # Restablecer el resultado
            self.reset()

        elif data in ("1", "2", "3", "4", "5", "6", "7", "8", "9", "0", ".", "+", "-", "×", "÷", "(", ")", "[", "]"):
            if self.result.value == "0" or self.new_operand:
                self.result.value = data
                self.new_operand = False
            else:
                self.result.value += data

        elif data == "=":
            self.result.value = str(self.calculate_expression(self.result.value))
            self.new_operand = True

        elif data == "%":
            self.result.value = str(float(self.result.value) / 100)
            self.new_operand = True

        elif data == "+/-":
            if self.result.value.startswith("-"):
                self.result.value = self.result.value[1:] # Quitar el signo negativo
            else:
                self.result.value = "-" + self.result.value # Agregar el signo negativo

        elif data == "√":
            self.result.value = str(math.sqrt(float(self.result.value)))   # Calcular la raíz cuadrada
            self.new_operand = True

        elif data == "x²":
            self.result.value = str(float(self.result.value) ** 2)
            self.new_operand = True

        self.update()

# Calcular el resultado de una expresión matemática.
    def calculate_expression(self, expression):
        try:
            expression = expression.replace("×", "*").replace("÷", "/")
            result = eval(expression)
            return self.format_number(result)
        except Exception as e:
            print(e)
            return "Error"

# Formatear el número para mostrarlo sin decimales si es un entero
    def format_number(self, num):
        if num % 1 == 0:
            return int(num)
        else:
            return num

    def reset(self):
        self.new_operand = True

# Clase para la página 1 de la aplicación, que muestra la calculadora
class Page1:
    def __init__(self):
 #  Inicializa la página 1 con una instancia de CalculatorApp
        self.calculator_app = CalculatorApp()

    def show_app_name(self):
        return self.calculator_app.display_name()


import flet as ft


# Clase para el apartado de resolución de problemas matemáticos
class MathProblemsApp(ft.Container):
    def __init__(self):
        super().__init__()

        self.problem_input = ft.TextField(
            hint_text="Busca problemas matemáticos",  # Texto de sugerencia en el campo de entrada
            expand=True,
            suffix=ft.IconButton(
                icon=ft.icons.CLOSE,  # Icono de cerrar para limpiar el texto
                icon_size=20,
                on_click=self.clear_text,
            ),
            text_align=ft.TextAlign.CENTER,
            text_style=ft.TextStyle(
                font_family="Georgia"
            )
        )

        self.problem_detected = False
        self.solution = ""  # Almacenar la solución del problema

        self.solution_text = ft.Text(
            value="",
            size=17,
            text_align=ft.TextAlign.CENTER,
            font_family="Georgia",
            visible=False  # Ocultar inicialmente
        )

        self.problem_not_recognized_text = ft.Text(
            value="",
            color=ft.colors.BLACK,
            size=17,
            text_align=ft.TextAlign.CENTER,
            font_family="Georgia",
            visible=False  # Ocultar inicialmente
        )

        self.solve_button = ft.ElevatedButton(
            text="SOLUCIÓN",  # Texto del botón para mostrar la solución
            on_click=self.show_solution,
            style=ft.ButtonStyle(
                text_style=ft.TextStyle(
                    size=20,
                    color=ft.colors.PURPLE,
                    font_family="Georgia"
                )
            ),
            visible=False  # Ocultar inicialmente
        )

        self.click_here_text = ft.Text(
            value="Clica aquí",
            size=18,
            color=ft.colors.PURPLE,
            font_family="Georgia",
            visible=False  # Ocultar inicialmente
        )

        self.problem_input.on_change = self.detect_problem  # Configurar la detección de problemas al cambiar el texto

        self.content = ft.Column(
            controls=[
                ft.Container(margin=ft.margin.only(top=20)),
                self.problem_input,
                ft.Container(
                    content=ft.Column(
                        controls=[
                            ft.Row(
                                controls=[self.click_here_text, self.solve_button],
                                alignment=ft.MainAxisAlignment.CENTER,
                            ),
                            ft.Container(margin=ft.margin.only(top=10)),
                            self.solution_text  # Mover el texto de solución aquí
                        ],
                        alignment=ft.MainAxisAlignment.CENTER,
                        spacing=10
                    ),
                    alignment=ft.alignment.center,
                ),
                self.problem_not_recognized_text
            ],
            spacing=10,
        )


    def clear_text(self, e):
        self.problem_input.value = ""
        self.solution = ""
        self.problem_not_recognized_text.value = ""
        self.solution_text.value = ""
        self.solve_button.visible = False
        self.click_here_text.visible = False
        self.solution_text.visible = False  # Ocultar texto de solución al borrar
        self.update()  # Actualizar la interfaz de usuario

    # Detectar si el texto del problema ingresado en el campo de entrada coincide con alguno de los problemas predefinidos.
    # Muestra u oculta el botón de solución y el texto de instrucción según si se ha detectado un problema o no.
    def detect_problem(self, e):
        self.problem_detected = False
        problem_text = self.problem_input.value.strip().lower()

        if problem_text:
            self.solve_button.visible = True
            self.click_here_text.visible = True
        else:
            self.solve_button.visible = False
            self.click_here_text.visible = False

        if "dos kilos de plátanos y tres de peras cuestan 7,80 euros" in problem_text:
            self.solution = """
        1. Ecuaciones:
           2x + 3y = 7,80
           5x + 4y = 13,20

        2. Resolución:
           x = 1,2 (plátanos)
           y = 1,8 (peras)
        """
            self.problem_detected = True

        elif "un fabricante de bombillas gana 0,3 euros por cada bombilla" in problem_text:
            self.solution = """
        1. Ecuaciones:
           x + y = 2100
           0,3x - 0,4y = 484,4

        2. Resolución:
           x = 1892 (buenas)
           y = 208 (defectuosas)
        """
            self.problem_detected = True

        elif "un agricultor tiene dos tipos de manzanas" in problem_text:
            self.solution = """
        1. Ecuaciones:
           x + y = 1200
           x = 2y

        2. Resolución:
           x = 800 (rojas)
           y = 400 (verdes)
        """
            self.problem_detected = True

        elif "encuentra las raíces de la ecuación cuadrática ax² + bx + c = 0" in problem_text:
            self.solution = """
        1. Ecuación cuadrática:
           x = (-b ± √(b² - 4ac)) / 2a

        2. Resolución:
           x1 = 3
           x2 = 2
        """
            self.problem_detected = True

        elif "resuelve la ecuación logarítmica log₂(x) + log₂(x - 3) = 3" in problem_text:
            self.solution = """
        1. Ecuación:
           x · (x - 3) = 2³

        2. Resolución:
           x = 4
        """
            self.problem_detected = True

        else:
            self.solution = "Problema no reconocido. Intente nuevamente."
            self.problem_detected = False

        self.update()  # Actualizar la interfaz de usuario para reflejar cambios


    def show_solution(self, e):
        if self.problem_detected:
            self.solution_text.value = self.solution
            self.solution_text.visible = True
        else:
            self.solution_text.value = ""
            self.solution_text.visible = False
        self.update()  # Actualizar la interfaz de usuario


# Crear un contenedor de fondo con una fila de iconos de estrellas para la interfaz
def background_rect():
    return ft.Container(
        bgcolor=ft.colors.BLUE_50,
        padding=0,
        height=60,  # altura del contenedor
        content=ft.Stack(
            controls=[
                ft.Row(
                    controls=[
                        ft.IconButton(
                            icon=ft.icons.LINE_WEIGHT_ROUNDED,   # Icono de estrella
                            icon_size=20,
                            style=ft.ButtonStyle(
                                color=ft.colors.CYAN_400,
                            ),
                        ) for _ in range(22)
                    ],
                    alignment=ft.MainAxisAlignment.CENTER,
                )
            ],
            alignment=ft.alignment.center,
        ),
    )

# Configuración de  la página principal de la aplicación con título, tema y pestañas
def main(page: ft.Page):
    page.title = "MATHBASICS"
    page.horizontal_alignment = ft.CrossAxisAlignment.CENTER
    page.vertical_alignment = ft.MainAxisAlignment.CENTER
    page.padding = 50
    page.theme_mode = ft.ThemeMode.LIGHT

    # Agregar el fondo rectángulo
    page.add(background_rect())

    title = ft.Text("MATHBASICS", size=33, color=ft.colors.PURPLE, font_family="Papyrus")

    # Instancia de las diferentes secciones de la aplicación
    page1 = Page1()
    math_problems_app = MathProblemsApp()
    microfono_tab = MicrofonoTab()
    graficos_tab = GraficosTab()

    page.add(ft.Column(controls=[title], alignment=ft.MainAxisAlignment.CENTER, spacing=20))

    # Configurar las pestañas de la aplicación
    t = ft.Tabs(
        selected_index=1,
        animation_duration=300,
        tabs=[
            ft.Tab(
                text=" RESOLUCIÓN DE PROBLEMAS MATEMÁTICOS",
                icon=ft.icons.SEARCH,
                content=ft.Container(
                    content=math_problems_app,
                    alignment=ft.alignment.center
                ),
            ),
            ft.Tab(
                text="CALCULADORA",
                icon=ft.icons.CALCULATE,
                content=ft.Container(
                    content=page1.calculator_app,
                    alignment=ft.alignment.center
                ),
            ),
            ft.Tab(
                text="MICRÓFONO",
                icon=ft.icons.MIC,
                content=ft.Container(
                    content=microfono_tab,
                    alignment=ft.alignment.center
                ),
            ),
            ft.Tab(
                text="GRÁFICAS DE ECUACIONES",
                icon=ft.icons.GRAPHIC_EQ,
                content=ft.Container(
                    content=graficos_tab,
                    alignment=ft.alignment.center
                ),
            ),
        ],
        expand=1,
    )

    page.add(t) #  # Agregar las pestañas a la página


ft.app(target=main)   # Ejecutar la aplicación con la función main como objetivo

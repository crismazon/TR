
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

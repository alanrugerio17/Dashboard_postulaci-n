library(shiny)
library(sf)
library(readxl)
library(dplyr)
library(leaflet)
library(viridis)
library(bslib)

malla <- st_read(
  "Datos/malla_hex_10km.gpkg",
  layer = "hex_10km",
  quiet = TRUE
)

malla$hex_id <- as.character(malla$hex_id)

archivos <- list.files(
  path = "Datos",
  pattern = "^temperatura_[0-9]{4}_ssp[0-9]+\\.xlsx$",
  full.names = TRUE,
  ignore.case = TRUE
)

if (length(archivos) == 0) {
  stop("No se encontraron archivos de temperatura en la carpeta Datos.")
}

leer_archivo <- function(archivo) {
  
  nombre <- tolower(basename(archivo))
  
  anio <- sub(
    "^temperatura_([0-9]{4})_ssp[0-9]+\\.xlsx$",
    "\\1",
    nombre
  )
  
  escenario <- sub(
    "^temperatura_[0-9]{4}_(ssp[0-9]+)\\.xlsx$",
    "\\1",
    nombre
  )
  
  read_excel(
    archivo,
    sheet = "datos"
  ) |>
    transmute(
      hex_id = as.character(hex_id),
      temperatura_c = as.numeric(temperatura_c),
      anio = as.integer(anio),
      escenario = toupper(escenario)
    )
}

datos <- bind_rows(
  lapply(archivos, leer_archivo)
)

escenarios <- sort(unique(datos$escenario))
estados <- sort(unique(na.omit(malla$estado)))

paleta <- colorNumeric(
  palette = c(
    "#FFFFB2",
    "#FED976",
    "#FEB24C",
    "#FD8D3C",
    "#FC4E2A",
    "#E31A1C",
    "#B10026"
  ),
  domain = datos$temperatura_c,
  na.color = "transparent"
)

ui <- fluidPage(
  theme = bs_theme(
    version = 5,
    bootswatch = "flatly"
  ),
  
  tags$head(
    tags$style(
      HTML("
        body {
          background-color: #f4f6f8;
        }

        .titulo-principal {
          margin-bottom: 20px;
        }

        .panel-lateral {
          background-color: white;
          padding: 20px;
          border-radius: 8px;
          box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
        }

        .panel-mapa {
          background-color: white;
          padding: 15px;
          border-radius: 8px;
          box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
        }

        .estadistica {
          margin-bottom: 12px;
          padding: 10px;
          background-color: #f7f9fa;
          border-left: 4px solid #2c3e50;
          border-radius: 4px;
        }

        .estadistica strong {
          display: block;
          font-size: 13px;
          color: #5d6d7e;
        }

        .estadistica span {
          font-size: 18px;
          color: #212529;
        }
      ")
    )
  ),
  
  div(
    class = "container-fluid",
    
    div(
      class = "titulo-principal",
      h2("Temperatura proyectada en México"),
      p("Escenarios climáticos SSP126, SSP370 y SSP585")
    ),
    
    fluidRow(
      column(
        width = 3,
        
        div(
          class = "panel-lateral",
          
          h4("Filtros"),
          
          selectInput(
            inputId = "escenario",
            label = "Escenario",
            choices = escenarios,
            selected = escenarios[1]
          ),
          
          selectInput(
            inputId = "anio",
            label = "Año",
            choices = NULL
          ),
          
          selectInput(
            inputId = "estado",
            label = "Estado",
            choices = c("Todos", estados),
            selected = "Todos"
          ),
          
          downloadButton(
            outputId = "descargar",
            label = "Descargar datos",
            class = "btn-primary"
          ),
          
          tags$hr(),
          
          h4("Resumen"),
          
          uiOutput("resumen"),
          
          tags$hr(),
          
          h4("Metadatos"),
          
          tags$p(
            tags$strong("Variable: "),
            "Temperatura"
          ),
          
          tags$p(
            tags$strong("Unidad: "),
            "Grados Celsius (°C)"
          ),
          
          tags$p(
            tags$strong("Resolución: "),
            "Malla hexagonal de 10 km"
          )
        )
      ),
      
      column(
        width = 9,
        
        div(
          class = "panel-mapa",
          
          h3(
            textOutput(
              outputId = "titulo_mapa",
              inline = TRUE
            )
          ),
          
          leafletOutput(
            outputId = "mapa",
            height = "720px"
          )
        )
      )
    )
  )
)

server <- function(input, output, session) {
  
  observeEvent(
    input$escenario,
    {
      anios <- datos |>
        filter(escenario == input$escenario) |>
        distinct(anio) |>
        arrange(anio) |>
        pull(anio)
      
      updateSelectInput(
        session = session,
        inputId = "anio",
        choices = anios,
        selected = anios[1]
      )
    },
    ignoreInit = FALSE
  )
  
  datos_seleccionados <- reactive({
    req(input$escenario, input$anio)
    
    datos |>
      filter(
        escenario == input$escenario,
        anio == as.integer(input$anio)
      )
  })
  
  mapa_completo <- reactive({
    malla |>
      left_join(
        datos_seleccionados(),
        by = "hex_id"
      )
  })
  
  mapa_filtrado <- reactive({
    req(input$estado)
    
    if (input$estado == "Todos") {
      mapa_completo()
    } else {
      mapa_completo() |>
        filter(estado == input$estado)
    }
  })
  
  output$titulo_mapa <- renderText({
    paste(
      "Temperatura",
      input$escenario,
      input$anio,
      input$estado,
      sep = " · "
    )
  })
  
  output$resumen <- renderUI({
    
    tabla <- mapa_filtrado() |>
      st_drop_geometry() |>
      filter(!is.na(temperatura_c))
    
    req(nrow(tabla) > 0)
    
    promedio <- mean(tabla$temperatura_c)
    minimo <- min(tabla$temperatura_c)
    maximo <- max(tabla$temperatura_c)
    hexagonos <- nrow(tabla)
    
    tagList(
      div(
        class = "estadistica",
        tags$strong("Temperatura promedio"),
        tags$span(paste0(round(promedio, 2), " °C"))
      ),
      
      div(
        class = "estadistica",
        tags$strong("Temperatura mínima"),
        tags$span(paste0(round(minimo, 2), " °C"))
      ),
      
      div(
        class = "estadistica",
        tags$strong("Temperatura máxima"),
        tags$span(paste0(round(maximo, 2), " °C"))
      ),
      
      div(
        class = "estadistica",
        tags$strong("Hexágonos con información"),
        tags$span(format(hexagonos, big.mark = ","))
      )
    )
  })
  
  output$mapa <- renderLeaflet({
    leaflet(
      options = leafletOptions(
        preferCanvas = TRUE
      )
    ) |>
      addProviderTiles(providers$CartoDB.Positron) |>
      setView(
        lng = -102,
        lat = 23.5,
        zoom = 5
      )
  })
  
  observe({
    
    tabla <- mapa_filtrado()
    
    req(nrow(tabla) > 0)
    
    mostrar_bordes <- input$estado != "Todos"
    limites <- st_bbox(tabla)
    
    proxy <- leafletProxy(
      mapId = "mapa",
      data = tabla
    ) |>
      clearShapes() |>
      clearControls() |>
      addPolygons(
        fillColor = ~paleta(temperatura_c),
        fillOpacity = 0.85,
        stroke = mostrar_bordes,
        color = "#5d6d7e",
        weight = 0.4,
        opacity = 0.7,
        smoothFactor = 0.5,
        popup = ~paste0(
          "<strong>Hexágono:</strong> ",
          hex_id,
          "<br>",
          "<strong>Estado:</strong> ",
          estado,
          "<br>",
          "<strong>Escenario:</strong> ",
          escenario,
          "<br>",
          "<strong>Año:</strong> ",
          anio,
          "<br>",
          "<strong>Temperatura:</strong> ",
          ifelse(
            is.na(temperatura_c),
            "Sin información",
            paste0(round(temperatura_c, 2), " °C")
          )
        )
      ) |>
      addLegend(
        pal = paleta,
        values = ~temperatura_c,
        title = "Temperatura (°C)",
        position = "bottomright",
        opacity = 0.85
      )
    
    proxy |>
      fitBounds(
        lng1 = limites[["xmin"]],
        lat1 = limites[["ymin"]],
        lng2 = limites[["xmax"]],
        lat2 = limites[["ymax"]]
      )
  })
  
  output$descargar <- downloadHandler(
    
    filename = function() {
      
      estado <- if (input$estado == "Todos") {
        "mexico"
      } else {
        tolower(gsub(" ", "_", input$estado))
      }
      
      paste0(
        "temperatura_",
        input$anio,
        "_",
        tolower(input$escenario),
        "_",
        estado,
        ".csv"
      )
    },
    
    content = function(file) {
      
      tabla <- mapa_filtrado() |>
        st_drop_geometry() |>
        select(
          hex_id,
          tipo,
          zona,
          estado,
          escenario,
          anio,
          temperatura_c
        )
      
      write.csv(
        tabla,
        file,
        row.names = FALSE,
        fileEncoding = "UTF-8"
      )
    }
  )
}

shinyApp(
  ui = ui,
  server = server
)
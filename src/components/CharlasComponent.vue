<template>
  <div>
    <div class="container">
      <!-- Filtro por ronda -->
      <div class="row row-cols-1 row-cols-md-2 mt-5">
        <div class="col">
          <label for="filtroRonda" class="form-label fw-semibold h5">Filtrar por ronda:</label>
          <select id="filtroRonda" class="form-select" v-model="filtroRonda" @change="filtrarCharlas">
            <option value="0" selected>Todas las rondas</option>
            <option v-for="ronda in rondas" :key="ronda.idRonda" :value="ronda.idRonda">
              {{ `Ronda ${ronda.idRonda} - ${ronda.descripcionModulo}` }}
            </option>
          </select>
        </div>

        <div class="col">
          <label for="filtroEstado" class="form-label fw-semibold h5">Filtrar por estado:</label>
          <select id="filtroEstado" class="form-select" v-model="filtroEstado" @change="filtrarCharlas">
            <option value="">Todos los Estados</option>
            <option v-for="estado in estadosDisponibles" :key="estado" :value="estado">
              {{ estado }}
            </option>
          </select>
        </div>
      </div>

      <div class="flex-container">
        <h5 style="margin-left: 18px;">Rondas</h5>
        <h5 style="margin-right: 20px;">Tiempo restante</h5>
      </div>
      <hr>

      <!-- Collapses de rondas -->
      <div class="accordion mt-4" id="accordionRondas">
        <div v-for="ronda in rondasFiltradas" :key="ronda.idRonda" 
        class="accordion-item">
        <h2 class="accordion-header d-flex justify-content-between align-items-center" :id="`heading-${ronda.idRonda}`">    
            <div class="container-btn-abrirModalRonda" v-if="perfilUser.idRole != 2">
              <i class="fa-solid fa-circle-info btn-abrirModalRonda" @click="abrirModalRonda(ronda.idRonda)"></i>  
            </div>
            <button class="accordion-button collapsed" type="button"
              data-bs-toggle="collapse" :data-bs-target="`#collapse-${ronda.idRonda}`" aria-expanded="false"
              :aria-controls="`collapse-${ronda.idRonda}`">
              <span>{{ `Ronda ${ronda.idRonda} - ${ronda.descripcionModulo}` }}</span>
              <div class="ms-auto" style="display: flex; flex-direction: row; align-items: center;">
                <div v-if="perfilUser.role != 'ALUMNO'"  class="info-ronda info-ronda-duracion rounded-5  text-center" style="width: 125px;">
                  <span class="text-muted">
                    <i class="fa-solid fa-clock-rotate-left" style="margin-right: 5px;"></i> {{ totalTiempoCharlas(ronda.idRonda) }} / {{ ronda.duracion }}min
                  </span>
                </div>
                <div class="info-ronda info-ronda-restante rounded-5 text-center" style="width: 125px;">
                  <span class="text-muted">
                    <i class="fa-regular fa-clock" style="margin-right: 5px;"></i> {{ obtenerTiempoRestante(ronda) }}
                  </span>
                </div>
              </div>
            </button>
          </h2>

          <div :id="`collapse-${ronda.idRonda}`" class="accordion-collapse collapse"
            :aria-labelledby="`heading-${ronda.idRonda}`" data-bs-parent="#accordionRondas">
            <div class="accordion-body">
              <div class="row g-3">
                <!-- Cards de charlas dentro de cada ronda -->
                <div class="col-xxl-4 col-xl-6 col-lg-6 col-md-12 mb-4" v-for="charla in charlasPorRonda(ronda.idRonda)" :key="charla.idCharla">
                  <div class="card">
                    <img class="card-img-top" :src="charla.imagenCharla ||
                      require('../assets/banner_default.jpg')
                      " @error="onImageError($event)" alt="Imagen de Charla" />
                    <div class="card-body">
                      <div class="d-flex justify-content-between align-items-center mb-4">
                        <p v-if="perfilUser.idRole != 1" className="m-0">
                          {{charla.estadoCharla === "ACEPTADA" ? `${charla.usuario.split(" ")[0]} - ` : ""}}🕒 {{charla.tiempo}} min
                        </p>
                        <p v-if="perfilUser.idRole != 2" class="m-0">{{charla.usuario.split(" ")[0]}} - 🕒 {{ charla.tiempo }} min</p>
                        <div class="d-flex justify-content-end gap-1">
                          <div v-if="charla.estadoCharla" :class="estadoClass(charla.estadoCharla)" class="card-badge">
                            {{ charla.estadoCharla }}
                          </div>
                          <div v-if="perfilUser.idRole != 2" class="btn btn-info card-badge votos">Votos:
                            {{ votosCharlas[charla.idCharla] ?? '0' }}
                          </div>
                        </div>                      
                      </div>
                      <h5 class="card-title">{{ charla.titulo }}</h5>
                      <p class="card-text">{{ charla.descripcion }}</p>
                    </div>
                    <div class="card-footer">
                      <small class="text-body-secondary d-flex justify-content-between align-items-center w-100">
                        <button style="margin-top: 0px;" class="btn custom-button" @click="abrirModal(charla)">
                          Ver detalles
                        </button>
                      </small>
                    </div>
                  </div>
                </div>
 
                <!-- Mensaje si no hay charlas -->
                <div v-if="charlasPorRonda(ronda.idRonda).length === 0" class="text-center text-muted">
                  No hay charlas disponibles para esta ronda.
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Modal para mostrar detalles -->
    <div v-if="mostrarModal" class="modal fade show" @click.self="cerrarModal" tabindex="-1" role="dialog"
      style="display: block; background: rgba(0, 0, 0, 0.8)">
      <div class="modal-dialog modal-lg modal-dialog-centered" role="document">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">{{ charlaSeleccionada.titulo }}</h5>
            <button type="button" class="btn-close" aria-label="Close" @click="cerrarModal"></button>
          </div>
          <div class="modal-body">
            <p class="timestamp">
              <strong>Fecha Propuesta:</strong>
              {{ formatearFecha(charlaSeleccionada.fechaPropuesta) }}
            </p>
            <p><strong>Usuario:</strong> {{ charlaSeleccionada.usuario }}</p>
            <p><strong>Curso:</strong> {{ charlaSeleccionada.nombreCurso }}</p>
            <p><strong>Estado:</strong> {{ charlaSeleccionada.estadoCharla }}</p>
            <p><strong>Duración:</strong> {{ charlaSeleccionada.tiempo }} minutos</p>            
            <!-- Botones para cambiar entre Descripción, Comentarios y Recursos -->
            <div class="d-flex custom-buttons-container">
              <button class="custom-button-detalles"
                @click="mostrarDescripcion = !mostrarDescripcion; mostrarComentarios = false; mostrarRecursos = false;"
                :class="{ 'active': mostrarDescripcion }">
                <i class="fa-solid fa-circle-info iconos"></i>
                Descripción
              </button>
              <button
                class="custom-button-detalles d-flex align-items-center"
                @click="
                  mostrarDescripcion = false;
                  mostrarComentarios = !mostrarComentarios;
                  mostrarRecursos = false;
                "
                :class="{ active: mostrarComentarios }"
              >
                <i class="fa-solid fa-comments iconos"></i>
                <span>Comentarios&nbsp;&nbsp; </span>
                <span v-if="comentarios.length > 0" class="comment-count">{{
                  comentarios.length
                }}</span>
              </button>
              <button
                class="custom-button-detalles d-flex align-items-center"
                @click="
                  mostrarDescripcion = false;
                  mostrarComentarios = false;
                  mostrarRecursos = !mostrarRecursos;
                "
                :class="{ active: mostrarRecursos }"
              >
                <i class="fa-solid fa-book iconos"></i>
                <span>Recursos &nbsp;&nbsp;</span>
                <span
                  v-if="recursos.length > 0"
                  class="count-badge"
                  :class="{ 'active-count': mostrarRecursos }"
                >
                  {{ recursos.length }}
                </span>
              </button>
            </div>

            <hr v-if="mostrarDescripcion || mostrarComentarios || mostrarRecursos" />
            <!-- Sección de Descripción -->
            <div v-if="mostrarDescripcion" class="custom-background custom-descripcion">
              <p>{{ charlaSeleccionada.descripcion }}</p>
            </div>

            <!-- Sección de Comentarios -->
            <div v-if="mostrarComentarios">
              <div v-if="comentarios.length > 0" class="custom-background">
                <ul class="comment-list">
                  <li v-for="comentario in comentarios" :key="comentario.idComentario" class="comment-item">
                    <div class="comment-header">
                      <img src="https://cdn.pixabay.com/photo/2015/10/05/22/37/blank-profile-picture-973460_640.png"
                        alt="avatar" class="avatar" />
                      <div>
                        <p class="username">{{ comentario.usuario }}</p>
                        <p class="timestamp">{{ comentario.fecha }}</p>
                      </div>
                    </div>
                    <hr />
                    <p class="comment-text">{{ comentario.contenido }}</p>
                  </li>
                </ul>
              </div>
              <div v-else>
                <p class="no-comments">No hay comentarios aún.</p>
              </div>
              <div class="comment-form">
                <textarea v-model="newComment" class="form-control" rows="3"
                  placeholder="Escribe tu comentario aquí..."></textarea>
                <button class="btn custom-button mt-2" @click="addComment">Agregar comentario</button>
              </div>
            </div>

            <!-- Sección de Recursos -->
            <div v-if="mostrarRecursos" class="custom-background">
              <div v-if="recursos.length > 0">
                <ul class="recurso-list">
                  <li v-for="recurso in recursos" :key="recurso.idRecurso" class="recurso-item">
                    <div class="recurso-header">
                      <h6 class="recurso-title">{{ recurso.nombre }}</h6>
                      <router-link :to="recurso.url" target="_blank" class="recurso-link">
                        <i class="fa-solid fa-link"></i> Ver Recurso
                      </router-link>
                    </div>
                    <p class="recurso-description">{{ recurso.descripcion }}</p>
                  </li>
                </ul>
              </div>
              <div v-else>
                <p class="no-recursos">No hay recursos disponibles.</p>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <!-- MODAL PARA LAS RONDAS -->
    <!-- Modal para mostrar de la Ronda -->
    <div v-if="mostrarModalRonda" class="modal fade show" @click.self="cerrarModalRonda" tabindex="-1" role="dialog"
      style="display: block; background: rgba(0, 0, 0, 0.8)">
      <div class="modal-dialog modal-lg modal-dialog-centered" role="document">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">
              {{ `Ronda - ${rondaActual.descripcionModulo}` }}
            </h5>
            <button type="button" class="btn-close" aria-label="Close" @click="cerrarModalRonda"></button>
          </div>
          <div class="modal-body">
            <p class="timestamp">
              <strong>Votos:</strong>
              {{votosRonda.votoscompletados}} / {{votosRonda.alumnoscurso}}
            </p>
            <p><strong>Charlas:</strong> {{charlasAceptadas.length + charlasPropuestas.length}}</p>
            <p><strong>Aceptadas:</strong> {{charlasAceptadas.length}}</p>
            <p><strong>Propuestas :</strong> {{charlasPropuestas.length}}</p>
            <div v-if="rolActual != 'ALUMNO'" class="d-flex custom-buttons-container">
              <router-link class="custom-button"
                :to="`/dragandrop/${rondaActual.idRonda}`">
                <i class="fa-solid fa-pen-to-square iconos"></i>
                Gestionar
              </router-link>
              <button class="custom-button"
                @click="mostrarNoVotados = !mostrarNoVotados;"
                :class="{ 'active': mostrarNoVotados }">
                <i class="fa-solid fa-user-minus iconos"></i>
                Usuarios sin votar
              </button>
            </div>

            <hr v-if="mostrarNoVotados" />
            <!-- Sección de Descripción -->
            <div  v-if="mostrarNoVotados && alumnosSinVotar.length > 0" class="custom-background custom-descripcion">
              <div v-for="alumno in alumnosSinVotar" :key="alumno.idUsuario">
                <p>{{ alumno }}</p>
              </div>
            </div>
            <div  v-if="mostrarNoVotados && alumnosSinVotar.length == 0" class="custom-background custom-descripcion">
              <p>Todos los alumnos han votado en esta ronda!</p>
            </div>
            </div>
          </div>
        </div>
      </div>
  </div>
</template>

<script>
import CharlasService from "@/services/CharlasService";
//NUEVO
import PerfilService from "@/services/PerfilService";
import Swal from "sweetalert2";
const serviceCharlas = new CharlasService();
import moment from 'moment';
import 'moment/locale/es';
const servicePerf = new PerfilService();
import Cookies from 'cookies-js';

export default {
  name: "CharlasComponent",
  data() {
    return {
      //NUEVO
      charlasPropuestas: [],
      charlasAceptadas: [],
      votosPropuestos: 0, // Total de votos de las charlas propuestas
      totalVotos: 0, // Total de votos posibles
      servicePerfil: new PerfilService(),
      charlasService: new CharlasService(),
      perfilUser: null,
      votosRonda: null, 
      votosCurso: null, 
      mostrarModalRonda: false, 
      //NUEVO
      charlas: [],
      rondas: [],
      comentarios: [],
      newComment: "",
      filtroRonda: 0,
      filtroEstado: "",
      charlasFiltradas: [],
      estadosDisponibles: [],
      recursos: [],
      rolActual: "ALUMNO",
      mostrarModal: false,
      charlaSeleccionada: null,
      mostrarDescripcion: false,
      mostrarRecursos: false,
      mostrarComentarios: false,
      votosCharlas: {},
      mostrarNoVotados: false,
      alumnosSinVotar: {}, 
      role: 0,
      rondaActual: {}
    };
  }, computed: {
    rondasFiltradas() {
      // Filtra las rondas según el filtro seleccionado
      if (this.filtroRonda == 0) {
        return this.rondas;
      } else {
        return this.rondas.filter(ronda => ronda.idRonda === this.filtroRonda);
      }
    }
  }, created() {
    this.rolActual = Cookies.get('user_role');
  },
  methods: {
    //NUEVO
    async cargarDatosCharlasRondas() {
        if (localStorage.getItem('perfilUser')) {
          try {
            this.perfilUser = 
            JSON.parse(localStorage.getItem('perfilUser'));
          } catch(e) {
            localStorage.removeItem('perfilUser');
          }
        }else{
          this.perfilUser = 
            await this.servicePerfil.getUsuarioPerfil();
          const parsed = JSON.stringify(this.perfilUser);
          localStorage.setItem('perfilUser', parsed)            
        }  
      if (this.perfilUser.idRole == 1){
        const alumnosPorCurso = await this.servicePerfil.getAlumnosActivoProfesor();
      this.totalVotos = alumnosPorCurso.length;
      this.charlasPropuestas = 
        this.charlas.filter((charla) => charla.idEstadoCharla === 1);
      this.charlasAceptadas = 
        this.charlas.filter((charla) => charla.idEstadoCharla === 2);
      const votosCharlas = 
        await this.charlasService.getVotosCurso(this.perfilUser.idCurso);
      for (const charla of this.charlasPropuestas) {
          const charlaConVotos = votosCharlas.filter((voto) => voto.idCharla === charla.idCharla);
          charla.votos = charlaConVotos.votos ?? 0;
          this.votosPropuestos += charla.votos; 
      }
      
      this.charlasPropuestas.sort((a, b) => b.votos - a.votos);

      // Procesar votos para charlas aceptadas
      for (const charla of this.charlasAceptadas) {
          const charlaConVotos = votosCharlas.filter((voto) => voto.idCharla === charla.idCharla);
          charla.votos = charlaConVotos.votos ?? 0;
          this.votosPropuestos += charla.votos; 
      }
      this.charlasAceptadas.sort((a, b) => b.votos - a.votos);     
      }
    },
    async votosPorRonda(idRonda) {
      this.votosRonda = 
        await this.charlasService.getVotosRonda(idRonda);
      let charlasFilter = this.charlas;
      charlasFilter = this.charlas.filter((charla) => charla.idRonda === idRonda);
      this.charlasPropuestas = 
        charlasFilter.filter((charla) => charla.idEstadoCharla === 1);
      this.charlasAceptadas = 
        charlasFilter.filter((charla) => charla.idEstadoCharla === 2);        
    },
    async votosPorCurso(idCurso) {
      this.votosCurso = 
        await this.charlasService.getVotosCurso(idCurso);
    },
    async votosPorCharla(idCharla) {
      var votos = 
        this.votosRonda.filter((ronda) => ronda.idCharla === idCharla);
      console.log(votos);
    },
    async alumnosNoVotaron(idRonda) {
      if(this.rolActual != 'ALUMNO'){
        this.alumnosSinVotar = await this.charlasService.getAlumnosSinVotoRonda(idRonda);
      }
    },
    async abrirModalRonda(idRonda) {
      this.rondaActual = await this.charlasService.getRonda(idRonda)
      await this.votosPorRonda(idRonda)
      this.mostrarModalRonda = true;
      await this.alumnosNoVotaron(idRonda);
    },
    cerrarModalRonda() {
      this.mostrarNoVotados = false;
      this.mostrarModalRonda = false;
    },
    abrirModal(charla) {
      this.charlaSeleccionada = charla;
      this.detalles(charla.idCharla);
      this.mostrarModal = true;
      this.cargarComentarios(charla.idCharla);
      this.cargarRecursos(charla.idCharla);
    },
    detalles(idCharla) {
      serviceCharlas.getDetallesCharla(idCharla)
        .then(response => {
          this.recursos = response.recursos;
        });
    },
    formatearFecha(fecha) {
      return moment(fecha).format('DD/MM/YYYY HH:mm');
    },
    cerrarModal() {
      this.mostrarModal = false;
      this.charlaSeleccionada = null;
      this.mostrarDescripcion = false;
      this.mostrarComentarios = false;
      this.mostrarRecursos = false;
    },
    cargarRondas() {
      serviceCharlas
        .getRondas()
        .then((response) => {
          this.rondas = response;
          this.cargarCharlas();
        })
        .catch((error) => {
          console.error("Error al cargar las rondas:", error);
        });
    },
    cargarRecursos(idCharla) {
      serviceCharlas
        .getRecursosCharlas(idCharla)
        .then((response) => {
          this.recursos = response.recursos || [];
          console.log(response.recursos)
        })
        .catch((error) => {
          console.error("Error al cargar los recursos:", error);
          Swal.fire("Error", "No se pudieron cargar los recursos.", "error");
        });
    },
    cargarCharlas() {
      serviceCharlas
        .getCharlas()
        .then((response) => {
          this.charlas = response;
          //console.log(this.charlas);
          this.charlasFiltradas = this.charlas;
          //console.log(response);
          this.filtrarCharlas();
          this.estadosDisponibles = [
            ...new Set(this.charlas.map((charla) => charla.estadoCharla)),
          ];
          const distinct = Object.values(
            this.charlas.reduce((o, a) => { o[a.idRonda] = a; return o}, {}));
          if (this.perfilUser.idRole == 2){
            //ALUMNO, COMPROBAMOS SUS VOTOS EN CADA RONDA
            let rondasVotadas = [];
            for (var ronda of distinct){
              this.charlasService.getVotoAlumnoRonda(ronda.idRonda).then(response => {
                  if (response.idUsuario != null){
                    //ALMACENAMOS LAS RONDAS DONDE HA VOTADO EL ALUMNO
                    rondasVotadas.push(response.idRonda);
                    let charlasOnRonda = this.charlas.
                      filter((charla) => charla.idRonda === response.idRonda);
                    charlasOnRonda.forEach(charla =>
                      this.cargarVotosCharla(charla.idCharla));
                  }
              })
            }
          }else {
            //PROFESOR O ADMIN
            this.charlas.forEach(charla =>
              this.cargarVotosCharla(charla.idCharla));
          }
        })
        .catch((error) => {
          console.error("Error al cargar las charlas:", error);
        });
    },
    filtrarCharlas() {
      this.charlasFiltradas = this.charlas.filter(charla => {
        const cumpleRonda = this.filtroRonda == 0 || charla.idRonda === this.filtroRonda;
        const cumpleEstado = this.filtroEstado === "" || charla.estadoCharla === this.filtroEstado;
        return cumpleRonda && cumpleEstado;
      });
    },
    charlasPorRonda(idRonda) {
      return this.charlasFiltradas.filter(
        (charla) => charla.idRonda === idRonda
      );
    },
    totalTiempoCharlas(idRonda) {
      return this.charlasPorRonda(idRonda)
      .filter(charla => charla.estadoCharla === "ACEPTADA") // Filtrar solo aceptadas
         .reduce((total, charla) => total + charla.tiempo, 0); // Sumar los tiempos
    },
    //Para una página de detalles
    mostrarDetalles(idCharla) {
      this.$router.push({ name: "DetallesCharla", params: { id: idCharla } });
    },
    estadoClass(estado) {
      switch (estado) {
        case "PROPUESTA":
          return "btn btn-gris";
        case "ACEPTADA":
          return "btn btn-success btn-verde";
        case "RECHAZADA":
          return "btn btn-danger";
        default:
          return "btn btn-info";
      }
    },
    onImageError(event) {
      event.target.src = require("../assets/banner_default.jpg");
    },
    //Para la carga y creacion de comentarios
    cargarComentarios(idCharla) {
      serviceCharlas
        .getCharlasComentarios(idCharla)
        .then((response) => {
          // Formatear las fechas de los comentarios al cargar
          this.comentarios = response.comentarios.map(comentario => {
            comentario.fecha = moment(comentario.fecha).locale('es').format('DD/MM/YYYY HH:mm');
            return comentario;
          });
        })
        .catch((error) => {
          console.error("Error al cargar los comentarios:", error);
          Swal.fire("Error", "No se pudieron cargar los comentarios.", "error");
        });
    },

    // Método para agregar un nuevo comentario
    addComment() {
      if (!this.newComment.trim()) {
        return;
      }
      const fechaActual = moment().locale('es').format('DD/MM/YYYY HH:mm'); 

      const comentario = {
        idCharla: this.charlaSeleccionada.idCharla,
        idUsuario: 13, 
        contenido: this.newComment,
        fecha: fechaActual,
      };

      this.isLoading = true; 

      serviceCharlas
        .setComentario(comentario)
        .then((response) => {
          console.log("Comentario creado:", response);
          Swal.fire({
            icon: "success",
            title: "Comentario agregado exitosamente!",
          });

          this.newComment = "";
          this.cargarComentarios(this.charlaSeleccionada.idCharla);
          this.isLoading = false;
        })
        .catch((error) => {
          this.isLoading = false;
          Swal.fire({
            icon: "error",
            title: "Error al agregar comentario",
            text: "No se pudo agregar el comentario. Revisa los datos enviados.",
          });
          console.error("Error al agregar el comentario:", error);
        });
    },
    obtenerTiempoRestante(ronda) {
      const ahora = new Date();
      const fechaPresentacion = new Date(ronda.fechaPresentacion);
      const fechaVotacion = new Date(ronda.fechaLimiteVotacion);
      const fechaCierre = new Date(ronda.fechaCierre);

      if (ahora < fechaPresentacion) {
        return `${this.formatearTiempo(fechaPresentacion - ahora)}`;
      } else if (ahora >= fechaPresentacion && ahora <= fechaCierre) {
        return `${this.formatearTiempo(fechaCierre - fechaVotacion)} - Vota ya!`;
      }
      return `Finalizado`;
    },
    formatearTiempo(ms) {
      const dias = Math.floor(ms / (1000 * 60 * 60 * 24));
      const horas = Math.floor(ms % (1000 * 60 * 60 * 24) / (1000 * 60 * 60));
      const minutos = Math.floor((ms % (1000 * 60 * 60)) / (1000 * 60));
      const segundos = Math.floor((ms % (1000 * 60)) / 1000);
      if (dias !== 0) {
        return `${dias} días`;
      }
      else {
        if (horas > 10)
          return `${horas} horas ${minutos} min`;
        if (horas < 10)
        return `${horas} hora ${minutos} min`;
        if (horas == 0)
          return `${minutos} min`;
        if (minutos < 0)
          return `${segundos} seg`;
      }
    },    
    async cargarVotosCharla(idCharla) {
      serviceCharlas.getVotosCharla(idCharla)
      .then(response => {
        this.votosCharlas[idCharla] = response.votos;
        //console.log(response);
      })
        .catch(() => {
        console.error("❌ No se pudieron obtener los votos de la charla ID", idCharla);
      });
    },
    obtenerUsuario() {
      servicePerf.getUsuarioPerfil()
      .then(response => {
        const parsed = JSON.stringify(response.usuario);
          localStorage.setItem('perfilUser', parsed)
        this.role = response.usuario.idRole;
        this.perfilUser = response.usuario;
      //console.log("rol: ", this.role);
      })
        .catch(() => {
        console.error("❌ No se pudo obtener el usuario");
      });
    },
    async getVotosAlumno() {
      await this.serviceCharlas.getVotosAlumno();
    },    
  },
  mounted() {
    this.cargarRondas(); // Cargar rondas al iniciar
    if (localStorage.getItem('perfilUser')) {
          try {
            this.perfilUser = 
              JSON.parse(localStorage.getItem('perfilUser'));
          } catch(e) {
            localStorage.removeItem('perfilUser');
          }
    }else{
          this.obtenerUsuario();
    } 
    this.cargarDatosCharlasRondas();
    //getVotosAlumno
  }
};
</script>

<style scoped>
.flex-container {
  margin-top: 40px;
  display: flex;
  justify-content: space-between;
}

.card-text {
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  line-clamp: 3;
  /* Líneas que se van a mostrar */
  -webkit-line-clamp: 2;
  /* Número de líneas visibles */
  -webkit-box-orient: vertical;
  padding: 5px 0px 5px 0px;
}

.card-img-top {
  width: 100%;
  border-radius: 24px 24px 0 0;
  max-height: 120px;
  object-fit: cover;
}

.container {
  margin-top: 20px;
}

.card-body {
  position: relative;
  height: 170px;
}

.card-badge {
  border-radius: 24px;
  font-size: 12px;
  color: black;
  margin: 0;
  padding: 2px 15px;
}

.card-header {
  font-weight: bold;
  background-color: #f8f9fa;
}

.card {
  box-shadow: 0 1px grey;
  /* border-bottom: solid 1px #494949; */
  transition: transform 0.3s;
  margin-bottom: 20px;
  margin-top: 10px;
  border-radius: 24px;
}

.card:hover {
  transform: translateY(-5px);
}

.estado-btn {
  position: absolute;
  top: 10px;
  left: 10px;
  padding: 5px;
  color: white;
  font-weight: bold;
  border-radius: 5px;
  z-index: 10;
  pointer-events: none;
}

.btn-verde {
  background-color: #87D0B1;
  border: none;
  cursor: default;
}

.btn-gris {
  background-color: #C6C6C6;
  border: none;
  cursor: default;
}

.votos {
  background-color: #C9DCFF;
  border: none;
  cursor: default;
}

.modal-body {
  padding: 15px 30px 15px 30px !important;
}

.custom-buttons-container {
  display: flex;
  /* Usamos flex para alinear los botones */
  align-items: center;
  /* Centrado vertical */
  justify-content: flex-start;
  /* Alineación a la izquierda */
  gap: 15px;
  /* Espaciado entre botones */
  margin-top: 25px;
}

.custom-button {
  border-radius: 20px;
  color: white;
  font-size: 15px;
  font-family: "Montserrat", serif;
  font-weight: 600;
  text-decoration: none;
  padding: 5px 20px;
  background-color: #4651C5;
  transition: background-color 0.3s ease, transform 0.2s ease;
  border: none;
}

.custom-button-detalles {
  border-radius: 20px;
  color: #3d3d3d;
  font-size: 15px;
  font-family: "Montserrat", serif;
  font-weight: 600;
  text-decoration: none;
  padding: 10px 20px;
  background-color: #f8f9fa;
  transition: background-color 0.3s ease, transform 0.2s ease;
  border: 1px solid #527c58;
}

.custom-button-detalles:hover {
  background-color: #527a5899;
  border-radius: 20px;
  color: #3d3d3d;
  /* Texto blanco en hover */
  font-family: "Montserrat", serif;
  /* Fuente consistente */
  font-weight: bold;
  transition: all 0.3s ease-in-out;
}

.custom-button:hover {
  cursor: pointer;
}

.custom-button.active {
  background-color: #527c58;
  /* Fondo en active */
  color: #fff;
}

.container-btn-abrirModalRonda {
  border-right: #888 1px dotted;
  padding-right: 10px;
  margin-right: 2px;
  padding-bottom: 5px;
}
.btn-abrirModalRonda {
  margin-left: 10px;
  font-size: 20px;
  color: #4651c5;
}
.btn-abrirModalRonda:hover {
  cursor: pointer;
}

.iconos {
  margin-right: 10px;
}

.modal-header {
  background-color: #7CA982;
}

.modal-title {
  font-family: "Montserrat", serif;
  font-weight: 600;
}

.custom-background {
  background-color: #eeeeee;
  padding: 20px;
  border-radius: 15px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  margin-top: 20px;
}

.custom-descripcion {
  font-size: 16px;
  color: #333;
  line-height: 1.6;
}

.comment-list {
  max-height: 400px;
  overflow-y: auto;
  padding-right: 10px;
  background-color: #d1e7d7;
  border-radius: 10px;
  margin-left: 0px !important;
}

/* Estilo de cada comentario */
.comment-item {
  background-color: #ffffff;
  /* Fondo blanco para los comentarios */
  padding: 15px;
  margin-bottom: 15px;
  margin-top: 15px;
  list-style-type: none;
  border-radius: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  /* Sombra más sutil */
  transition: transform 0.3s ease-in-out;
  color: #333;
  /* Color del texto más oscuro para asegurar buena visibilidad */
}

.comment-item:hover {
  background-color: #f1f1f1;
  /* Fondo más claro al pasar el mouse */
}


.no-comments {
  font-size: 16px;
  color: #888;
  text-align: center;
  margin-top: 10px;
}

.comment-header {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin-right: 15px;
  border: 2px solid #888;
  /* Espacio entre la imagen y el texto */
}

.username {
  font-weight: bold;
  font-size: 14px;
  color: #333;
  /* Color más oscuro para el nombre */
}

.timestamp {
  font-size: 12px;
  color: #888;
  margin-top: 5px;
}

.comment-text {
  font-size: 14px;
  color: #555;
  margin-top: 10px;
}

.no-comments {
  font-size: 16px;
  color: #888;
  text-align: center;
  margin-top: 10px;
}

/* Estilo para el formulario de agregar comentario */
.comment-form {
  margin-top: 20px;
}

.comment-form textarea {
  width: 100%;
  resize: none;
  padding: 10px;
  border-radius: 8px;
  border: 1px solid #ccc;
  font-size: 14px;
  color: #333;
  background-color: #f1f1f1;
  margin-bottom: 10px;
}

.comment-form button {
  width: 100%;
  padding: 12px;
  background-color: #527c58;
  /* Fondo del botón acorde con los otros botones */
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: bold;
  transition: background-color 0.3s ease;
}

.comment-form button:hover {
  background-color: #406b45;
  /* Fondo más oscuro al hacer hover */
}

.accordion-button::after {
  margin-left: 28px !important;
}

/* Estilo para la sección de recursos */

.recurso-list {
  list-style-type: none;
  padding: 0;
}

.recurso-item {
  background-color: #d1e7d7;
  padding: 20px;
  margin-bottom: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease-in-out;
}

.recurso-item:hover {
  transform: translateY(-8px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.recurso-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.recurso-title {
  font-size: 18px;
  font-weight: bold;
  color: #527c58;
  margin: 0;
}

.recurso-link {
  font-size: 14px;
  color: #527c58;
  text-decoration: none;
  display: flex;
  align-items: center;
}

.recurso-link:hover {
  text-decoration: underline;
  color: #406b45;
}

.recurso-link i {
  margin-right: 5px;
}

.recurso-description {
  font-size: 14px;
  color: #555;
  margin-top: 10px;
}

.no-recursos {
  font-size: 16px;
  color: #888;
  text-align: center;
  margin-top: 10px;
}

.accordion-button:not(.collapsed) {
  background-color: #E2F3FF;
}

.info-ronda {
  padding: 5px 10px;
  font-size: 15px;
  border-bottom: #33333367 2px solid;
}

.info-ronda-duracion {
  background-color: rgb(188, 218, 230);
  margin-right: 15px;
}
.info-ronda-restante {
  background-color: rgb(183, 219, 183);
}
/* BOTONES DETALLES CONTADOR COMENTARIOS Y RECURSOS */
.custom-button-detalles {
  display: flex;
  align-items: center;
  gap: 8px; /* Espaciado entre icono, texto y contador */
}

.comment-count {
  background-color: #4651c5;
  color: white;
  font-size: 12px;
  font-weight: bold;
  width: 35px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 20px;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* Cambia el color del contador cuando el botón está activo */
.custom-button-detalles.active .comment-count {
  background-color: white;
  color: #494949;
}

.count-badge {
  background-color: #4651c5;
  color: white;
  font-size: 12px;
  font-weight: bold;
  width: 35px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 20px;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* Cambia el color del contador cuando el botón está activo */
.custom-button-detalles.active .count-badge {
  background-color: white;
  color: #494949;
}

.custom-button-detalles.active {
  background-color: #527a5899;
}
</style>

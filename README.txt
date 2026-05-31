THERMAL PRINTER SERVER
======================

Este paquete contiene la aplicación Thermal Printer Server compilada para Windows x64.
La aplicación es completamente portable e incluye todas sus dependencias internamente. No necesitas instalar ningún entorno de ejecución previo.

INSTRUCCIONES RÁPIDAS
---------------------
1. Haz doble clic en "manage.bat" para abrir el panel de administración.
2. Desde allí podrás iniciar el servidor manualmente, o instalarlo como un servicio en segundo plano de Windows.

¡ADVERTENCIA IMPORTANTE SOBRE LA REUBICACIÓN DE ESTA CARPETA!
-------------------------------------------------------------
La aplicación en sí es portable, pero si decides instalarla como un Servicio de Windows (Tarea Programada), el sistema operativo registrará la ruta absoluta donde se encuentra instalada en este preciso momento.

Si en el futuro decides MOVER esta carpeta a otro directorio u otro disco, el servicio se corromperá y dejará de funcionar en silencio.
Para evitar esto o corregirlo, debes seguir estos pasos:
  1. Abre "manage.bat" en la ubicación actual y desinstala el servicio.
  2. Mueve la carpeta a su nueva ubicación.
  3. Abre "manage.bat" en la nueva ubicación y vuelve a instalar el servicio.

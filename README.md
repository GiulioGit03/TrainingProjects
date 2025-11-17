# void OrdinaPalline(Pallina[] palline, int numeroColori)
{
    // 1. Conta occorrenze per colore
    int[] conteggi = new int[numeroColori];

    foreach (var p in palline)
        conteggi[p.Colore]++;

    // 2. Ricostruisci l’array ordinato
    int index = 0;
    for (int colore = 0; colore < numeroColori; colore++)
    {
        int count = conteggi[colore];
        for (int i = 0; i < count; i++)
        {
            palline[index++] = new Pallina { Colore = colore };
        }
    }
}
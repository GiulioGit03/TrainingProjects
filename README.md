private static void AddValue(Node root, int value)
{
    // Se root è null in teoria dovresti assegnare al tree.Root,
    // ma visto che questo metodo viene chiamato solo con tree.Root già creato,
    // gestiamo solo i figli.

    if (value < root.Value)
    {
        // Vai a sinistra
        if (root.Left == null)
        {
            root.Left = new Node(value);
        }
        else
        {
            AddValue(root.Left, value);
        }
    }
    else if (value > root.Value)
    {
        // Vai a destra
        if (root.Right == null)
        {
            root.Right = new Node(value);
        }
        else
        {
            AddValue(root.Right, value);
        }
    }
    else
    {
        // Value già presente — non fare nulla (dipende da progetto)
    }
}
public static Node? FindValue(Node root, int value)
{
    if (root == null)
        return null;

    if (value == root.Value)
        return root;

    if (value < root.Value)
        return FindValue(root.Left, value);

    return FindValue(root.Right, value);
}

Re-seed admin after any docker compose down -v:


docker exec linguist--postgres-1 psql -U linguist -d linguist << 'EOF'
INSERT INTO "User" (username, email, password, is_admin)
VALUES ('admin', 'admin@example.com', 'abc123', TRUE)
ON CONFLICT (username) DO UPDATE SET is_admin = TRUE;
EOF